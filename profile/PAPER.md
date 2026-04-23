# NEURA: A Policy-Gated Delegated Agent with Bounded Web Orchestration

## 1. Abstract

This repository implements a **dual-path conversational backend** built on FastAPI. One path exposes a **delegated-tool agent** (`/agent/run`, `/agent/continue`) that routes each user turn through a **deterministic ordered policy** over tool versus direct-answer choices, optional **LLM fallback** when the policy abstains, and **HTTP-executed tools** (`search_web`, `fetch_url`, weather tools, `run_code`). For multi-page web answers, `fetch_url` can invoke **server-side orchestration** that ranks search hits, fetches several URLs, extracts lightweight structure, scores conflicts, and runs a **bounded reconciliation controller** before synthesis. A second path, **`/chat/v2`**, is a separate **multi-pass LLM orchestrator** (fast / balanced / deep) with memory read/write and streaming; it does **not** embed the delegated web orchestrator described here. The UI can operate in “agent mode” (delegated loop) or standard chat (`/chat/v2`). Unless stated otherwise, claims below are grounded in the Python modules under `agent/`, `main.py`, `tools/` (where applicable), and `ui/src/`.

---

## 2. System Overview

**Delegated agent path (agent mode).** A user message is handled by `POST /agent/run`, which calls `run_agent` in `agent/agent_loop.py`. The pipeline is:

1. **Memory context bundle** — `get_memory_context_bundle` builds text injected into the assistant system prompt (scoped by identity and memory set).
2. **Ordered policy** — `decide_action` in `agent/decision_policy.py` walks `POLICY_RULES` and returns `action` ∈ {`answer`, `tool`, `abstain`}.
3. **Early exit** — On `answer`, the server either evaluates trivial arithmetic locally or calls `call_llm` once and returns `{"type": "final", "output": ...}`.
4. **Tool request** — On `tool`, the server returns `{"type": "tool_request", "tool": ..., "input": ...}` for the **client** to execute (e.g. `POST /tools/search_web`, `POST /tools/fetch_url`).
5. **Continuation** — The client posts the tool result to `POST /agent/continue` → `continue_agent`, which normalizes the tool payload, prompts the model with `CONTINUE_AFTER_TOOL_SYSTEM_PROMPT`, and either returns another `tool_request` or a `final` string. Loop repetition on the same tool/input is **blocked** with a forced summarization path.

**Web orchestration (subset of tools).** When `fetch_url` is called with `search_results_json` and `user_query`, `main.py` delegates to `orchestrate_fetch_from_search` in `agent/delegated_web_orchestrator.py` (non-streaming or streaming via `POST /tools/fetch_url/stream`). That function performs search-result ranking, bounded multi-fetch, structured extraction, optional **multi-source synthesis** and **`run_reconciliation_controller`**.

**Standard chat path.** `POST /chat/v2` in `main.py` implements a different pipeline (multi-pass cynical/planner/final with SSE). It is **not** the same as `run_agent` and does not call `orchestrate_fetch_from_search` in the code inspected for this paper.

**UI.** `ui/src/App.tsx` chooses between `/chat/v2` and `/agent/run` + `/agent/continue` depending on agent mode, handles `tool_request`, optional auto tool execution, orchestrated `fetch_url` streaming, and `needs_user_continue` payloads.

---

## 3. Architecture

The codebase separates concerns as follows:

| Layer | Role | Primary modules |
|-------|------|-----------------|
| **Decision** | Ordered `POLICY_RULES`; first match wins; optional abstain + LLM JSON tool decision | `agent/decision_policy.py`, `agent/agent_loop.py` |
| **Orchestration** | Rank DDG results, fetch N pages, cross-source scoring, optional synth + reconcile | `agent/delegated_web_orchestrator.py`, `main.py` (`/tools/fetch_url`) |
| **Fact extraction** | Regex/heuristic structured fields; SPO-style fact **candidates** | `extract_structured_info`, `extract_structured_fact_candidates` in `delegated_web_orchestrator.py` |
| **Evaluation** | `evaluate_answer`, `score_answer_confidence`, cross-source agreement | Same file |
| **Memory** | SQLite graph, embeddings, retrieval bundles, domain success/failure logs | `agent/memory_graph.py`, `agent/memory_retrieval.py`, `agent/agent_memory_turn.py`, `agent/agent_domain_stats.py` |
| **Planner (lightweight)** | String list `build_plan` for debug metadata only | `agent/agent_loop.py` |
| **Planner (orchestration)** | Mutable `orchestration_plan` dict with named steps and budgets | `build_orchestration_plan`, `plan_mark` in `delegated_web_orchestrator.py` |
| **UI** | React app: streaming chat, agent state machine, tool runner | `ui/src/App.tsx` |

There is **no** separate global “planner LLM” inside `run_agent`; planning for web tasks is **data-structure updates** inside the orchestrator when orchestration runs.

---

## 4. Decision System

### 4.1 `POLICY_RULES`

`POLICY_RULES` in `agent/decision_policy.py` is a **fixed-order** list of rules. Each rule has a `condition` callable, an `action` (`answer` or `tool`), optional `tool` name for tool actions, and a `confidence` float. The list order is:

1. `trivial_math` → `answer`
2. `memory_direct_fetch` → `tool` / `fetch_url` when memory suggests a direct React doc URL and the agent allows `fetch_url`
3. `weather_factual` → `tool` / `weather` or `weather_query` / `weather_query_multi` (resolved in `build_decision_from_rule` via `ctx.weather_delegation`)
4. `avoid_tool` → `answer` for conceptual/opinion-style queries (heuristics in `is_conceptual_or_opinion`)
5. `force_tool` → `tool` / `search_web` when `requires_external_data` (forced tool spec from `PolicyContext.forced_tool_spec`)

```153:187:agent/decision_policy.py
POLICY_RULES: list[dict[str, Any]] = [
    {
        "name": "trivial_math",
        "condition": trivial_math_condition,
        "action": "answer",
        "confidence": 1.0,
    },
    {
        "name": "memory_direct_fetch",
        "condition": memory_direct_fetch_condition,
        "action": "tool",
        "tool": "fetch_url",
        "confidence": 0.95,
    },
    {
        "name": "weather_factual",
        "condition": is_weather_factual,
        "action": "tool",
        "tool": "weather",
        "confidence": 0.98,
    },
    {
        "name": "avoid_tool",
        "condition": is_conceptual_or_opinion,
        "action": "answer",
        "confidence": 0.9,
    },
    {
        "name": "force_tool",
        "condition": requires_external_data,
        "action": "tool",
        "tool": "search_web",
        "confidence": 0.9,
    },
]
```

Note: the static `"tool": "weather"` entry is superseded in `build_decision_from_rule` for `weather_factual`, which copies the actual tool name from `ctx.weather_delegation` (e.g. `weather_query_multi`).

### 4.2 `decide_action`

`decide_action` constructs `PolicyContext`, optionally computes `memory_bias_rule` (only tightens confidence for `weather_factual` when recent memory mentions weather), then **returns on the first rule whose condition is true**. If none match, it returns `action: "abstain"`.

```256:301:agent/decision_policy.py
def decide_action(
    *,
    query: str,
    agent: RegisteredAgent,
    cfg: AppConfig,
    identity: dict[str, Any] | None = None,
    memory_set: str | None = None,
    memory_graph: MemoryGraph | None = None,
) -> dict[str, Any]:
    """
    Walk ``POLICY_RULES`` in order; first matching rule wins.
    ...
    """
    ctx = PolicyContext(...)
    bias = memory_bias_rule(query, memory_graph)
    for rule in POLICY_RULES:
        ...
        if cond(ctx):
            dec = build_decision_from_rule(rule, ctx)
            ...
            return dec
    return {
        "action": "abstain",
        ...
    }
```

### 4.3 `run_agent` integration

`run_agent` calls `decide_action` before any LLM tool-decision prompt. If `action == "answer"`, it uses trivial math or `call_llm`. If `action == "tool"`, it validates the tool against `merged_tools()` and agent allowlists, then returns a `tool_request`. If the tool was policy-forced (`force_tool_policy`, `weather_factual`, `memory_direct_fetch`), a debug invariant can assert the result is `tool_request`.

If `action == "abstain"`, `run_agent` issues a **separate** LLM call with `TOOL_DECISION_SYSTEM_PROMPT`, parses JSON via `try_parse_tool_decision`, and may still return a `tool_request` or a final natural-language answer.

### 4.4 Tool forcing vs avoidance

Forcing uses `_force_tool_signals` in `decision_policy.py` (combines `agent_loop.should_force_tool`, `query_requires_external_tool`, and a small keyword list). Avoidance uses `is_conceptual_or_opinion`, which can return true for `agent_loop.should_avoid_tool`, policy `_conceptual_avoid_explanation`, or educational weather queries (`is_weather_educational_or_theory_query`). **Rule order** ensures weather and trivial math win before generic avoid/force.

---

## 5. Orchestration Engine

### 5.1 Entry points

- **Search:** `POST /tools/search_web` in `main.py` fetches DuckDuckGo HTML, extracts visible text and structured `results` via `extract_ddg_result_snippets`.
- **Single URL:** `POST /tools/fetch_url` without `search_results_json` calls `run_webfetch_logic` and returns markdown (truncated).
- **Orchestrated multi-source:** Same endpoint with `search_results_json` + `user_query` calls `orchestrate_fetch_from_search` with identity, memory set, optional `extended_search`, `memory_graph`, and `synthesize_llm=call_llm` from config.

### 5.2 `orchestrate_fetch_from_search` (high level)

The function:

1. Builds **`extract_memory_strategy_signals`** (bounded booleans and host lists; see §10).
2. Sets **`collect_target`** (base 3, +1 capped at 4 if `prefer_multi_source`) and **`max_pool_refetch_budget`** (base 2, +1 capped at 3 if `prefer_deeper_reconciliation`).
3. Ranks `results` via **`rank_search_results`**, which uses **`_search_result_score`** (query overlap, domain stats, preferred URL, memory hosts, strategy signal bumps/penalties).
4. Builds **`build_orchestration_plan`** and updates step status via **`plan_mark`** through the run.
5. Fetches up to `max_attempts` (3 or 6 if `extended_search`) pool URLs until `collect_target` or exhaustion; skips blocked pages (`is_blocked`).
6. If at least two collected sources and combined confidence threshold met, may call **`run_reconciliation_controller`** when `synthesize_llm` is provided; else concatenates summaries via **`weighted_synthesis`** path for markdown assembly.
7. Returns a dict shaped like a normal fetch (including `structured` with orchestration metadata when multi-source succeeds) or `{"type": "needs_user_continue", ...}`.

### 5.3 Ranking and domain statistics

`_search_result_score` combines:

- `_base_rank_score` on title/snippet/url token overlap with the query,
- **`domain_score_adjustment`** and **`domain_knowledge_rank_boost`** from `agent_domain_stats.py` (per identity/memory set/domain),
- preferred URL match bonus,
- memory-derived trusted domains and `source_trust` edge hosts,
- optional **`strategy_signals`** trusted/avoid host adjustments.

### 5.4 Extraction

Per fetched page, **`extract_structured_info`** produces `summary`, `numbers`, `numeric_values`, `dates`, `entities`, `key_facts`, and a heuristic `confidence`. There is **no** second-pass LLM for extraction in this module.

---

## 6. Fact Modeling System

### 6.1 `extract_structured_fact_candidates`

Defined in `agent/delegated_web_orchestrator.py`, this builds a list of dictionaries with keys including:

- `subject` — normalized string from **`infer_subject_candidates`** (entities, summary proper-noun regex, key-fact prefixes) with per-line **`_pick_subject_for_line`**,
- `predicate` — from **`infer_predicate_value_pairs`** (regexes for mayor, dimensional height/length/…, toll/cost/population/temperature, or a generic “X is Y” fallback),
- `value` — raw string slice,
- `value_type` — `number` | `date` | `entity` | `text` via **`_value_type_for_predicate`**,
- `source_index`, `source_field`.

Additional rows attach **`numeric_values`** as `predicate: "numeric_literal"` and **`dates`** as `predicate: "date"` tied to a default subject.

### 6.2 Normalization

**`normalize_entity_name`** lowercases, strips punctuation to spaces, removes a leading “the ”, truncates.

**`infer_predicate_value_pairs`** is explicitly regex-driven; it is **not** a dependency parser or NER model.

### 6.3 Entity handling

**`extract_entities`** uses capitalized phrase regexes over a bounded window of text. **`infer_subject_candidates`** dedupes normalized subjects from entities, summary, and key facts.

---

## 7. Conflict Detection

### 7.1 Numeric conflicts

**`detect_numeric_conflicts`** compares per-source numeric meaning sets derived from `numeric_values` or, if absent, **`extract_numeric_values`** on the summary. Pairs of sources with disjoint non-empty meaning sets produce conflict rows (`source_i`, `source_j`, value lists).

### 7.2 Semantic / summary disagreement

**`detect_contradictions`** compares **summary** texts pairwise with **`semantic_similarity`** (Jaccard over length-4+ tokens); low similarity yields a contradiction entry. This is separate from structured fact conflicts.

### 7.3 Structured fact conflicts

**`detect_fact_conflicts`** enumerates **`extract_structured_fact_candidates`** per source index, then pairs facts across sources where **`_subject_aligns`** and **`_predicate_aligns`** hold and normalized values differ (numeric tolerance via parsed floats; string inequality otherwise). Output rows include `subject`, `predicate`, `value_i`, `value_j`, and a `kind` among `mayor_mismatch`, `height_mismatch`, or `spo_mismatch`.

### 7.4 Agreement scoring

**`score_fact_agreement`** builds normalized triple signatures per fact (`subject`, `predicate`, normalized value), computes Jaccard-like overlap of signature sets between source pairs, averages over pairs, and returns a value in [0, 1] with a **default 0.55** when no comparable pairs exist (documented in code).

### 7.5 Use in synthesis

**`weighted_synthesis`** attaches `numeric_conflicts`, `fact_conflicts`, and `fact_agreement` to its return dict. **`synthesize_sources`** injects conflict descriptions into the user prompt for the synthesis LLM when `weighted_bundle` is supplied.

---

## 8. Reconciliation Controller

**`run_reconciliation_controller`** (`agent/delegated_web_orchestrator.py`) implements a **bounded** loop:

- **Inputs:** `collected`, `pool`, `urls_tried`, `user_query`, `synthesize_llm`, memory/identity for pool ordering, `max_extra_pool_fetch`, `max_synth_llm_passes`, optional `orc_plan`, optional `strategy_signals`.
- **Clamping:** `MAX_EXTRA_POOL_FETCH = max(1, min(max_extra_pool_fetch, 4))`, `MAX_SYNTH_LLM_PASSES` similarly capped at 4, `MAX_CONTROLLER_ITERS = 6`.
- **Flow:** Marks `reconcile_conflicts` in progress on the plan; computes **`weighted_synthesis`**; calls **`synthesize_sources`**; evaluates **`evaluate_answer`** on a bundle derived from weighted outputs + cross-source agreement + averaged confidence.
- **While** evaluation status is `needs_refinement` and iteration count `< MAX_CONTROLLER_ITERS`:
  - If reasons include numeric/fact/low agreement/low confidence conflicts **and** fetch budget remains, **`_extend_collected_for_reconciliation`** fetches up to `fetch_budget` additional pool URLs (ordered by `_sorted_pool_for_reconciliation`), appends trace notes (`RECONCILIATION_FETCH`, `NUMERIC_RECONCILIATION_FETCH`), may mark plan step `fetch_additional_reconciliation_sources`, re-synthesizes if synth passes remain.
  - Else if synth passes remain and reasons include numeric/fact/heavy contradiction, performs a **second LLM synthesis** with a fixed user suffix instructing explicit reconciliation.
  - Otherwise **breaks**.

Returns **`(markdown, weighted_dict, trace_notes)`**.

---

## 9. Evaluation System

**`evaluate_answer`** inspects a structured dict for:

- Non-empty `numeric_conflicts` → `needs_refinement`, reason `numeric_conflict`
- Non-empty `fact_conflicts` → `fact_conflict`
- `fact_agreement` &lt; 0.34 → `low_fact_agreement`
- `cross_source_agreement` &lt; 0.22 → `low_cross_source_agreement`
- `final_confidence` &lt; 0.52 → `low_final_confidence`
- `contradictions` list length ≥ 2 → `heavy_contradiction`

The controller maps these reasons to **extra fetches** or **extra LLM synthesis** as described in §8. **`orchestrate_fetch_from_search`** separately gates entering multi-source synthesis on an average confidence + cross-source agreement threshold (0.5) before invoking the controller.

---

## 10. Memory System

### 10.1 `MemoryGraph`

`agent/memory_graph.py` implements a SQLite-backed store with optional remote embeddings (Ollama/OpenAI), optional sentence-transformers, optional FAISS (`MemoryFaissIndex`), node metadata (`memory_type`, `trust_level`, token ids), and graph operations used across retrieval and persistence. Exact retrieval ranking blends multiple signals (embedding similarity, recency, trust, etc.); the full scoring algebra is **long**—this paper does not reproduce every branch; readers should consult `memory_graph.py` and `memory_retrieval.py` for formulas.

### 10.2 Context injection (delegated agent)

`run_agent` calls **`get_memory_context_bundle`** to produce `context` text included in the assistant system prompt. Follow-up turns can **`enrich_query_with_recent_context`** when **`is_follow_up`** is true.

### 10.3 Sensitive filtering

`enrich_query_with_recent_context` and related paths consult **`memory_hit_is_sensitive`** and **`is_explicit_sensitive_recall`** from `agent/sensitive_memory.py` to **skip** injecting sensitive memory content unless the user query explicitly recalls it.

### 10.4 Memory strategy signals (orchestration only)

**`extract_memory_strategy_signals`** returns a small dict: `prefer_multi_source`, `prefer_deeper_reconciliation`, `trusted_hosts`, `avoid_hosts`, `topic_bias`. Query substrings and recent memory text drive booleans and host extraction with **fixed list sizes**. These signals adjust **collect/refetch caps** (bounded), **`rank_search_results`** scoring, and reconciliation pool ordering—**not** `POLICY_RULES` order.

### 10.5 Influence on ranking and synthesis

Besides strategy signals, **`_memory_trusted_domains`** and **`_memory_trust_hosts_from_edges`** feed `_search_result_score`. **`_memory_synthesis_trust_lines`** prepends soft “prefer these hosts” guidance into the **`synthesize_sources`** user prompt when `memory_graph` is enabled.

### 10.6 `token_memory.py`

This module provides **token-id-oriented** helpers (packing, trimming, detokenize for debug, splicing). It complements the graph but is not the primary routing surface for the delegated agent; orchestration does not import it directly.

### 10.7 Domain stats

`agent_domain_stats` functions log successes/failures per domain and feed rank boosts—used both in orchestration ranking and in **`memory_suggested_direct_fetch_url`** (React doc shortcut) in `agent_memory_turn.py`.

---

## 11. Planner Model

### 11.1 Agent-loop `build_plan`

**`build_plan(query)`** in `agent/agent_loop.py` returns a **string list**: either `["search multiple sources", "fetch top results", "compare data", "synthesize"]` when the query contains compare/vs/versus, else `["search", "fetch", "answer"]`. It is attached to debug payloads via **`_maybe_debug`** when `AGENT_DELEGATED_AGENT_DEBUG` is enabled. It does **not** drive tool execution.

### 11.2 Orchestration plan object

**`build_orchestration_plan`** returns a dict: `goal`, `steps` (each `name` + `status`), `events` (list), and `budgets` (`max_collect`, `max_pool_refetch`, `max_synth_passes`). Extra steps `group_sources_by_subject` and `compare_fact_sets` appear when `deep_compare` is true (query heuristics and/or `prefer_multi_source` from signals). **`plan_mark`** mutates step rows in place. **`plan_steps_from_orchestration_plan`** flattens steps for API/debug. **`orchestrate_fetch_from_search`** stores the plan under `structured["orchestration_plan"]` and merges flattened steps plus `events` into `structured["orchestration_steps"]`.

### 11.3 Plan vs execution

Step statuses are updated when corresponding phases run (e.g. `fetch_sources` after the fetch loop, `reconcile_conflicts` / `final_synthesis` inside the controller). Early exits (insufficient sources, low confidence) mark later steps **`skipped`** with reasons in the orchestrator code path.

---

## 12. UI / Interaction Model

From **`ui/src/App.tsx`** (behavior inferred from symbols and comments in the file):

- **Agent state machine** includes states such as `thinking`, `waiting_for_continue`, and handling of streaming passes for `/chat/v2`.
- **`tool_request`**: When `autoRunTools` is false, the UI can require an explicit user continue; when true, it runs tools in a loop until a final message.
- **`needs_user_continue`**: Special-cased to preserve UI affordances for “continue searching” style flows; can surface extended search.
- **Orchestrated `fetch_url`:** **`runFetchUrlOrchestratedWithProgress`** prefers **`/tools/fetch_url/stream`**, parses SSE `progress` then `complete`, and falls back to non-streaming `POST /tools/fetch_url` if streaming fails.
- **Streaming:** `/chat/v2` consumes SSE chunks into `streaming` state; `LiveBubble` renders in-progress final-pass text.
- **Markdown:** Message bodies use chat styling; streamed content is shown in a `<pre>` for the live bubble path (implementation detail in `App.tsx`).
- **Debug:** `pushAgentDebug` records payloads such as `needs_user_continue` when debug toggles apply.

The UI does **not** implement policy or orchestration logic; it reflects API types.

---

## 13. End-to-End Execution Trace

**Example query:** `"compare weather in LA vs NYC"`.

**Important:** For factual weather comparisons, **`is_weather_tool_query`** in `agent/weather_openmeteo.py` returns true when `"weather"` co-occurs with a `vs` / `versus` pattern (among other conditions). **`_build_weather_delegation`** in `agent/agent_loop.py` then uses **`extract_locations`** (LLM-backed in `weather_openmeteo.py`) and **`parse_time_intent`** to build a payload. If the agent profile allows **`weather_query_multi`**, **`decide_action`** matches **`weather_factual`** before **`force_tool`**.

1. **HTTP:** `POST /agent/run` with the default delegated agent profile (`agent/agents.py` includes `weather_query_multi`).
2. **`run_agent`:** loads `MemoryGraph` via `graph_for_set`, builds memory context, calls **`decide_action`**. Assuming delegation succeeds, `action` is `tool` with tool `weather_query_multi` and structured input `{locations, time}` (exact keys from `_build_weather_delegation`).
3. **Client tools:** UI or script calls `POST /tools/weather_query_multi` → Open-Meteo wrapper in `main.py` → `weather_query_multi` in `weather_openmeteo.py`.
4. **`continue_agent`:** Tool result JSON is normalized (`normalize_tool_result_for_llm` with a higher char cap for weather). The continuation LLM summarizes; no `search_web` / `fetch_url` orchestration is **required** by policy for this path.

**If** `extract_locations` failed (returned empty), `weather_delegation` could be `None`; then **`weather_factual`** would not match. Subsequent rules may route to **`avoid_tool`** or **`force_tool`** depending on heuristics; **`synthesize_forced_tool`** would retry weather delegation or fall back to **`search_web`**. That branch is **conditional** on runtime LLM output—this paper does not assume it always happens.

**Contrast:** Had the query been a **non-weather** comparison requiring web pages, **`force_tool`** could emit `search_web`; the client would run search, then **`fetch_url`** with `search_results_json` + `user_query`, triggering **`orchestrate_fetch_from_search`** and possibly **`run_reconciliation_controller`**. That is the path where **SPO fact candidates**, **numeric/fact conflicts**, and **orchestration_plan** updates apply.

---

## 14. Design Principles

1. **Deterministic control before LLMs:** Policy order is fixed; numeric trivial answers bypass the model; forced-tool invariants can be asserted in debug mode.
2. **Bounded self-correction:** Reconciliation iterations, synth passes, and extra fetches are **explicitly capped**; same-tool repeats in `continue_agent` are detected and short-circuited.
3. **Separation of concerns:** Tool HTTP runs in `main.py`; agent reasoning in `agent_loop.py`; web multi-source logic isolated in `delegated_web_orchestrator.py`; memory persistence in `memory_graph.py`.
4. **Safety via constraints:** Sensitive memory gating; blocked-page detection; truncation on fetch markdown; tool-result normalization size limits in `continue_agent`.
5. **Memory as bounded influence:** Memory adjusts scores and small budgets in orchestration; **`memory_bias_rule`** only nudges confidence for a specific policy match and does not reorder rules.

---

## 15. Limitations

- **Heuristic extraction:** Summaries, entities, key facts, and SPO candidates are regex/Jaccard-based; errors are expected on messy HTML or implicit subjects.
- **LLM variability:** Location extraction for weather, tool-decision JSON after abstain, continuation tool choice, and synthesis text are all **model-dependent** despite deterministic scaffolding.
- **Search quality:** `search_web` depends on DuckDuckGo HTML scraping and snippet parsing quality.
- **Policy coverage:** `abstain` + LLM path can still request disallowed tools (then filtered) or miss tools; recovery depends on `synthesize_forced_tool` and continuation prompts.
- **Two chat architectures:** Behavior of `/chat/v2` vs `/agent/run` differs; operators must know which path the UI selected.
- **Incomplete formal verification:** The repository ships tests (e.g. `tests/test_delegated_web_orchestrator.py`) but no machine-checked proofs of policy or bounds.

---

## 16. Future Work

Grounded **only** in obvious gaps relative to current code:

- **Entity linking:** `infer_subject_candidates` and `_subject_aligns` are shallow; richer linking would require new algorithms or data, not present today.
- **Deeper memory influence:** Strategy signals already exist; expanding them would require new safeguards to preserve “no policy override” invariants tested conceptually in code comments and caps.
- **Long-horizon planning:** `build_orchestration_plan` adds named steps for compare-style queries but does not schedule arbitrary DAGs; extending plans would mean extending `build_orchestration_plan` / `plan_mark` usage, not `run_agent`.
- **Multimodal integration:** Current delegated tools and orchestrators are text/HTML-centric; there is **no** image/audio tool path in the inspected orchestrator.

---

## References (source files)

| Topic | Path |
|-------|------|
| FastAPI surface | `main.py` |
| Delegated loop | `agent/agent_loop.py` |
| Policy | `agent/decision_policy.py` |
| Web orchestration | `agent/delegated_web_orchestrator.py` |
| Tool catalog/registry | `agent/delegated_tool_catalog.py`, `agent/delegated_tool_registry.py` |
| Weather | `agent/weather_openmeteo.py` |
| Memory graph | `agent/memory_graph.py` |
| Memory in prompts / persistence | `agent/agent_memory_turn.py` |
| Sensitive gating | `agent/sensitive_memory.py` |
| Token-oriented memory utils | `agent/token_memory.py` |
| UI | `ui/src/App.tsx` |
| Tests (orchestrator) | `tests/test_delegated_web_orchestrator.py` |
