## NEURA
> orchestrates multiple independent AI models into a controlled reasoning pipeline that generates hypotheses gathers real world evidence from tools and memory graphs detects contradictions scores competing claims and deterministically resolves them into the most defensible conclusion before returning an answer. With self improvement.

`question` **→** `multiple models` **→** `tools` **+** `memory` **→** `arbitration` **→** `answer`

### Task Flow
```
(input task)
→ structured plan graph
→ step execution (per node)
→ multi-model reasoning (per step)
→ tools (build, logs, code, web)
+ memory (past fixes, patterns)
→ claim extraction (what’s wrong / what to change)
→ evidence gathering (tool outputs + memory)
→ truth arbitration (pick most likely fix)
→ action (apply change in sandbox)
→ validation (rebuild / tests)
→ final result
```
