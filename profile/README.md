## NEURA
> Policy driven AI middleware and agent system with deterministic routing structured tool orchestration SPO based fact extraction crosssource conflict detection agreement scoring bounded reconciliation loops memory guided strategy bias and execution aware planning transforming LLM outputs into a controlled self correcting verifiable reasoning and decision engine across dynamic real world tasks

`user query` **→** `policy decision engine` **→** `tool routing` **→** `multi-source orchestration` **→** `structured extraction (SPO + numerics + entities)` **→** `conflict detection (numeric + semantic)` **→** `evaluation + reconciliation controller` **→** `memory guided synthesis` **→** `final answer`


# [PAPER.md](https://github.com/neura-asi/.github/blob/main/profile/PAPER.md)


```  The architecture, mapped to concrete components

  gpt-oss-20b (patched llama-server, n-cpu-moe)   companion 1.5B (jlens, PyTorch)
          │ residual @ layer L (real)                    │ residual @ layer L
          ▼                                              ▼
    in-engine LOGIT LENS  ──────────┐         JACOBIAN LENS (future dir)
    (current belief, entropy)       │                    │
                                    ▼                    ▼
                          ┌──────  STATE ESTIMATOR (cogstate.py)  ──────┐
                          │  EMA/Kalman-lite smoothing over layers      │
                          └──────────────────┬──────────────────────────┘
                                             ▼
                              TRAJECTORY ANALYSIS
               entropy · layer-agreement(KL) · convergence · hypothesis tracks ·
                          current-vs-future divergence (uncertainty)
                                             ▼
                LIVE UI (dashboard)      +      ADAPTIVE ORCHESTRATION
          token-by-token cognition panel     (soft triggers: verify / retrieve /
                                              tool / early-stop — opt-in, gated)

```

# Runtime

Unlike traditional agent architectures that treat the language model as a black box, Neura continuously observes, estimates, and reasons about the model's evolving cognitive state.

```mermaid
flowchart TB

%% =====================================================
%% INPUT
%% =====================================================

U([User])
API[Neura Runtime]
MEM[(Adaptive Memory<br/>Knowledge Graph<br/>Repair Learning)]
CTX[Context Compiler<br/>Rehydration Engine]

U --> API
MEM --> CTX
CTX --> API

%% =====================================================
%% MODEL ORCHESTRATION
%% =====================================================

subgraph MODELS["Model Layer"]

direction LR

LOCAL["Neura-OSS-20B<br/>Patched llama.cpp"]
REMOTE["GPT-5.5 / Frontier Model"]
COMP["Companion 1.5B<br/>Jacobian Lens"]

end

API --> LOCAL
API --> REMOTE

%% =====================================================
%% RESIDUAL STREAM
%% =====================================================

subgraph RESIDUAL["Residual Stream Observer"]

direction TB

L0["Embedding"]
L1["Layer 1"]
L2["Layer 2"]
L3["..."]
LN["Final Layer"]

L0 --> L1 --> L2 --> L3 --> LN

end

LOCAL --> RESIDUAL

%% =====================================================
%% LOGIT LENS
%% =====================================================

subgraph LOGLENS["Native Logit Lens"]

direction TB

LOGITS["Current Belief"]
ENT["Entropy"]
TOPK["Top-K Tokens"]
PROBS["Probability Distribution"]

LOGITS --> ENT
LOGITS --> TOPK
LOGITS --> PROBS

end

%% =====================================================
%% JLENS
%% =====================================================

subgraph JLENS["Companion Jacobian Lens"]

direction TB

JRES["Residual Snapshot"]
JPRED["Future Direction"]
JCONF["Future Confidence"]

JRES --> JPRED
JPRED --> JCONF

end

RESIDUAL --> LOGLENS
RESIDUAL --> JRES
COMP --> JLENS

%% =====================================================
%% TOKEN GRAPH
%% =====================================================

subgraph TOKENGRAPH["Hierarchical Token Graph"]

direction TB

TOK["Token Nodes"]
EMB["Embedding Nodes"]
CON["Concept Nodes"]
HYP["Hypothesis Nodes"]
REL["Semantic Relationships"]

TOK --> EMB
EMB --> CON
CON --> HYP
HYP --> REL

end

LOGLENS --> TOKENGRAPH
JLENS --> TOKENGRAPH

%% =====================================================
%% GRAPH ANALYTICS
%% =====================================================

subgraph GRAPHOPS["Graph Analytics"]

direction TB

CENT["Centrality"]
COMM["Communities"]
PATH["Shortest Paths"]
PERSIST["Temporal Persistence"]
GRAPHENT["Graph Entropy"]

CENT --> GRAPHENT
COMM --> GRAPHENT
PATH --> GRAPHENT
PERSIST --> GRAPHENT

end

TOKENGRAPH --> GRAPHOPS

%% =====================================================
%% TRAJECTORY
%% =====================================================

subgraph TRAJ["Trajectory Engine"]

direction TB

VEL["Velocity"]
ACC["Acceleration"]
CURV["Curvature"]
DRIFT["Semantic Drift"]
ATTR["Attractors"]

VEL --> ACC
ACC --> CURV
CURV --> DRIFT
DRIFT --> ATTR

end

RESIDUAL --> TRAJ

%% =====================================================
%% COGNITIVE STATE
%% =====================================================

subgraph COGSTATE["Cognitive State Estimator"]

direction TB

OBS["Observation Fusion"]

EMA["EMA"]

KAL["Kalman-lite"]

BAYES["Bayesian Updates"]

POST["Posterior State"]

OBS --> EMA
OBS --> KAL
OBS --> BAYES

EMA --> POST
KAL --> POST
BAYES --> POST

end

LOGLENS --> OBS
JLENS --> OBS
GRAPHOPS --> OBS
TRAJ --> OBS

%% =====================================================
%% METRICS
%% =====================================================

subgraph METRICS["Inference Metrics"]

direction TB

CONF["Confidence"]

UNC["Uncertainty"]

CONV["Convergence"]

KL["Layer KL"]

AGREE["Lens Agreement"]

STAB["Belief Stability"]

ENT2["Entropy"]

POST --> CONF
POST --> UNC
POST --> CONV
POST --> KL
POST --> AGREE
POST --> STAB
POST --> ENT2

end

%% =====================================================
%% HYPOTHESIS TRACKING
%% =====================================================

subgraph HYPTRACK["Hypothesis Evolution"]

direction TB

BIRTH["Birth"]

GROW["Growth"]

MERGE["Merge"]

SPLIT["Split"]

DECAY["Decay"]

FINAL["Final Decision"]

BIRTH --> GROW
GROW --> MERGE
MERGE --> SPLIT
SPLIT --> DECAY
DECAY --> FINAL

end

POST --> HYPTRACK

%% =====================================================
%% ORCHESTRATION
%% =====================================================

subgraph ORCH["Adaptive Orchestration"]

direction TB

VERIFY["Verification"]

RETRIEVE["Memory Retrieval"]

TOOLS["Tool Invocation"]

REROUTE["Model Routing"]

EARLY["Early Exit"]

MORE["Request More Reasoning"]

VERIFY
RETRIEVE
TOOLS
REROUTE
EARLY
MORE

end

CONF --> ORCH
UNC --> ORCH
CONV --> ORCH
AGREE --> ORCH

%% =====================================================
%% MEMORY
%% =====================================================

subgraph MEMORY["Persistent Cognitive Memory"]

direction TB

STATE["World State"]

USERMEM["User Memory"]

REPO["Repository State"]

REPAIR["Repair Motifs"]

GRAPHMEM["Knowledge Graph"]

TIMELINE["Reasoning Timeline"]

STATE --> GRAPHMEM
USERMEM --> GRAPHMEM
REPO --> GRAPHMEM
REPAIR --> GRAPHMEM
TIMELINE --> GRAPHMEM

end

ORCH --> MEMORY
HYPTRACK --> MEMORY
TOKENGRAPH --> MEMORY

MEMORY --> CTX

%% =====================================================
%% OUTPUT
%% =====================================================

subgraph OUTPUT["Presentation Layer"]

direction TB

UI["Live Cognition Dashboard"]

HEAT["Layer Heatmaps"]

GRAPHVIEW["Semantic Graph"]

TIMELINE2["Reasoning Timeline"]

TOKVIEW["Token Evolution"]

METRICVIEW["Confidence / Entropy"]

UI --> HEAT
UI --> GRAPHVIEW
UI --> TIMELINE2
UI --> TOKVIEW
UI --> METRICVIEW

end

POST --> OUTPUT

%% =====================================================
%% FINAL RESPONSE
%% =====================================================

REMOTE --> RESP([Final Response])
LOCAL --> RESP
ORCH --> RESP
RESP --> U
```
