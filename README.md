# ΛXÖN

### Multi-Agent DAG Orchestrator

> *From a single iterating loop to a directed acyclic graph of specialised skills. The agent plans the graph; the executor runs it; nodes that share no dependencies run in parallel.*

[![Python](https://img.shields.io/badge/Python-3.11+-blue)](https://python.org)
[![NetworkX](https://img.shields.io/badge/Graph-NetworkX-green)](https://networkx.org)
[![LLM](https://img.shields.io/badge/LLM-Gateway%20V8-purple)](src/agent/llm_gateway/)
[![FAISS](https://img.shields.io/badge/Memory-FAISS%20768d-orange)](https://github.com/facebookresearch/faiss)

---

**ΛXÖN** is a DAG-based multi-agent orchestrator built as Session 8 of the [EAG V3](https://theschoolof.ai) program. It extends the Session 7 single-loop agent (Memory → Perception → Decision → Action) into a growing directed acyclic graph where a Planner decomposes queries into skill nodes, an Executor runs them concurrently via `asyncio.gather`, and a Critic verifies outputs with tool-grounded measurements.

**What makes it different from Session 7:**
- Parallel execution — three independent sub-tasks run concurrently, not sequentially
- Explicit graph structure — the trace is a diagram, not a list of iterations
- Per-skill scoping — each node sees only its relevant inputs, not cumulative history
- Tool-grounded critic — verification uses actual measurement tools, not LLM judgment
- Resumable execution — kill at any point, resume from the last completed node

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER QUERY                                     │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ░░ MEMORY.READ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░    │
│  FAISS vector search (768-d embeddings)                                     │
│  Returns: top-k ranked hits visible to ALL skill nodes this run             │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ◆◆ PLANNER ◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆◆          │
│  Reads: query + memory hits                                                 │
│  Outputs: JSON DAG — skill nodes with typed inputs and dependencies         │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  ★★ EXECUTOR ★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★★       │
│  Runs ready nodes concurrently · Persists after each · Handles failures      │
│                                                                              │
│  Available Skills:                                                           │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                        │  │
│  │  researcher     web_search, fetch_url — fetches from the web           │  │
│  │  retriever      search_knowledge — queries FAISS index                 │  │
│  │  coder          writes Python → sandbox_executor runs it               │  │
│  │  critic         count_syllables, count_characters — tool verification  │  │
│  │  shell          run_command — executes system commands                 │  │
│  │  fact_checker   web_search × 2 — confirms/disputes claims              │  │
│  │  comparator     ranks and selects from multiple inputs                 │  │
│  │  distiller      extracts structured fields from text                   │  │
│  │  summariser     condenses long content                                 │  │
│  │                                                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  Failure policy:                                                             │
│    transient → skip · validation → skip · upstream → recovery planner        │
│  Critic fail:                                                                │
│    skip child node → queue re-plan with different approach                   │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ◈◈ FORMATTER ◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈◈        │
│  Terminal node — renders upstream outputs as markdown answer                │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ▓▓ PERSISTENCE ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓       │
│  state/sessions/<sid>/graph.json + nodes/*.json + traces.jsonl              │
│  Atomic writes (tmp + os.replace) · Resume from any checkpoint              │
└─────────────────────────────────────────────────────────────────────────────┘
```





---

## Skill Catalogue

Skills are defined in `agent_config.yaml` — one YAML entry + one prompt file per skill. No Python class per skill. Adding a skill never touches the orchestrator.

| Skill | Temperature | Tools | Purpose |
|-------|-------------|-------|---------|
| **planner** | 0.4 | — | Decomposes queries into DAG nodes |
| **researcher** | 0.7 | web_search, fetch_url, create_file | Fetches content from the web |
| **retriever** | 0.3 | search_knowledge, read_file | Searches FAISS knowledge base |
| **distiller** | 0.1 | — | Extracts structured fields from text |
| **summariser** | 0.3 | — | Condenses long content |
| **critic** | 0.0 | count_syllables, count_characters | Pass/fail verification with tools |
| **formatter** | 0.3 | — | Renders final user-facing answer |
| **coder** | 0.2 | — | Writes Python for computation |
| **sandbox_executor** | 0.0 | — | Runs coder's Python in isolation |
| **comparator** | 0.2 | — | Compares and ranks multiple items |
| **fact_checker** | 0.2 | web_search | Verifies claims (confirmed/disputed/unverifiable) |
| **shell** | 0.2 | run_command | Executes system commands |

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Planner emits graph; Executor runs it | Model writes the program; runtime executes it. Same machinery for all skills. |
| Per-node scoping via `metadata.question` | Fan-out workers see only their slice, not the full multi-item query. Prevents "all researchers search for everything." |
| Researcher fast path (no LLM mediation) | Direct Tavily search + extract. Saves 2 LLM round-trips per researcher node. |
| Critic with tool access | `count_characters` and `count_syllables` ground verification in measurement, not LLM guessing. |
| Contract validation with retry | Coder must return `{"code": "..."}` or fail. One retry with error feedback before failing loudly. |
| Atomic graph persistence | `os.replace` after every node. Kill at any point → resume from last checkpoint. |
| Recovery classification | `transient` (skip), `validation_error` (skip), `upstream_failure` (re-plan). Prevents infinite recovery loops. |
| MAX_RECOVERY = 3 | Caps total re-plan attempts per run. Prevents runaway token spend. |
| Shell fast path | One LLM call to decide commands, then direct execution. No multi-turn tool loop overhead. |

---


## Recovery and Failure Handling

```
Node fails
    │
    ▼
classify_failure(error_text)
    │
    ├── "transient" (503, timeout, connection)
    │       → skip (gateway already retried)
    │
    ├── "validation_error" (malformed JSON, schema)
    │       → skip (prompt bug, not runtime)
    │
    └── "upstream_failure" (everything else)
            → queue recovery planner (max 3 per run)
                → planner emits new sub-graph
                    → executor continues
```

**Critic-fail splice:**
- Critic emits `{"verdict": "fail"}` → child node is marked "skipped"
- Recovery planner queued with failure rationale
- Recovery planner must choose a different approach (or cap hit → stop)

---

## Persistence and Resume

```
state/sessions/<sid>/
    query.txt              user query verbatim
    graph.json             NetworkX node_link_data (full DAG state)
    traces.jsonl           per-node span log (timing, tokens, provider)
    nodes/
        n_1.json           NodeState for planner
        n_2.json           NodeState for researcher
        ...
```

All writes are atomic (`write-tmp` + `os.replace`). Resume reads `graph.json`, resets `running` nodes to `pending`, continues the executor loop. Nodes already `complete` are not re-run.

---

## New Skills (Assignment Requirements)

### Shell Skill
Executes system commands via `run_command` MCP tool. Genuinely new capability — no existing skill can run `find`, `grep`, `wc`, `git`, `du`, etc.

**Demo query:** `"How many Python files are in this project and what's the total line count?"`

**Architecture:** shell → formatter (2 nodes). Shell fast path: 1 LLM call to decide commands, then direct `subprocess.run`.

### Fact Checker Skill
Verifies claims against web evidence. Searches both supporting and contradicting sources, returns structured verdict with URLs.

**Demo query:** `"Is it true that the Great Wall of China is visible from space with the naked eye?"`

**Architecture:** fact_checker → formatter (2 nodes). Exactly 2 searches (one for, one against), then verdict.

---

## Live Dashboard

Real-time WebSocket dashboard at `http://localhost:8080`:

- **SVG DAG graph** — nodes colored by status, edges show data flow
- **Grouped live logs** — Memory → Perception → Action → Decision per node
- **Parallel fan-out banners** — concurrent nodes nested in a group
- **Node I/O tab** — exact inputs/outputs passed between nodes
- **Metrics panel** — wall-clock, serial cost, speedup, tokens, cache hits
- **Session replay** — load past sessions with full graph and logs
- **Stop/Resume** — kill execution, persist state, continue later

---

---

## Query Traces & Evidence

> All screenshots below show actual ΛXÖN dashboard output with live DAG graph, grouped logs, and Node I/O.

### Base Queries

| Query | Description | Nodes |
|-------|-------------|-------|
| A | Shannon Wikipedia (fetch + extract) | 3 |
| I | Populations parallel fan-out (London, Paris, Berlin) | 6 |
| J | Graceful failure (/nonexistent/path.txt) | 2 |
| K | Resume (kill mid-run + resume from checkpoint) | 6 |

### Custom Queries

| Query | Description | Skills Used |
|-------|-------------|-------------|
| 1 | Multi-skill chain (LOC count + web compare) | shell, researcher, comparator, formatter |
| 2 | NVIDIA analysis (price + investment + verification) | researcher, coder, sandbox, fact_checker, formatter |
| 3 | Critic FAIL + recovery (140-char tweet) | formatter, critic, coder, sandbox |
| 4 | Coder + Sandbox (compound interest) | coder, sandbox_executor, formatter |

<details><summary><b>Click to expand: Query A — Shannon Wikipedia</b></summary>

-------

![A-1](screenshots/shannon-q-a-a.png)
![A-2](screenshots/shannon-q-a-b.png)
![A-3](screenshots/shannon-q-a-c.png)

</details>

<details><summary><b>Click to expand: Query I — Parallel Fan-out (Populations)</b></summary>

-------

![I-1](screenshots/parallel-q-i-a.png)
![I-2](screenshots/parallel-q-i-b.png)
![I-3](screenshots/parallel-q-i-c.png)

</details>

<details><summary><b>Click to expand: Query J — Graceful Failure</b></summary>

-------

![J-1](screenshots/gracefulfailure-q-j-a.png)
![J-2](screenshots/gracefulfailure-q-j-b.png)

</details>

<details><summary><b>Click to expand: Query K — Resume</b></summary>

-------

![K-1](screenshots/resume-q-k-a.png)
![K-2](screenshots/resume-q-k-b.png)
![K-3](screenshots/resume-q-k-c.png)
![K-4](screenshots/resume-q-k-d.png)
![K-5](screenshots/resume-q-k-e.png)

</details>

<details><summary><b>Click to expand: Custom Query 1 — Multi-skill Chain</b></summary>

-------

![C1-1](screenshots/Custom-multiskill-q-1a.png)
![C1-2](screenshots/Custom-multiskill-q-1b.png)
![C1-3](screenshots/Custom-multiskill-q-1c.png)
![C1-4](screenshots/Custom-multiskill-q-1d.png)
![C1-5](screenshots/Custom-multiskill-q-1e.png)

</details>

<details><summary><b>Click to expand: Custom Query 2 — NVIDIA Multi-skill Analysis</b></summary>

-------

![C2-1](screenshots/Custom-multiskill-nvidia-q-2a.png)
![C2-2](screenshots/Custom-multiskill-nvidia-q-2b.png)
![C2-3](screenshots/Custom-multiskill-nvidia-q-2c.png)
![C2-4](screenshots/Custom-multiskill-nvidia-q-2d.png)

</details>

<details><summary><b>Click to expand: Custom Query 3 — Critic FAIL + Recovery</b></summary>

-------

![C3-1](screenshots/critic-q-3a.png)
![C3-2](screenshots/critic-q-3b.png)
![C3-3](screenshots/critic-q-3c.png)
![C3-4](screenshots/critic-q-3d.png)

</details>

<details><summary><b>Click to expand: Custom Query 4 — Coder + Sandbox</b></summary>

-------

![C4-1](screenshots/coder-q-4a.png)
![C4-2](screenshots/coder-q-4b.png)

</details>


-------


## Setup & Running

### Prerequisites
- Python 3.11+
- [uv](https://docs.astral.sh/uv/) (recommended) or pip

### Installation

```bash
# Clone
git clone https://github.com/Shwethaamrutha/EAGv3-Session8.git
cd EAGv3-Session8

# Install dependencies
uv sync

# Or with pip
pip install -e .

# Configure environment
cp .env.example .env
# Edit .env with your API keys:
#   NVIDIA_API_KEY=...
#   GEMINI_API_KEY=...
#   TAVILY_API_KEY=...
#   AWS_REGION=us-east-1
```

### Running the Dashboard

```bash
cd src
python ../dashboard/server.py
# Open http://localhost:8080
```

### CLI Mode

```bash
cd src
python flow.py "Find the populations of London, Paris, Berlin and tell me which two are closest in size"

# Resume a killed session
python flow.py --resume s8_XXXXXXXX
```

### Run Tests

```bash
python -m pytest tests/test_recovery.py -v
```

---

## File Structure

```
agent8/
├── flow.py                 DAG orchestrator (Executor, Graph)
├── skills.py               Skill catalogue loader
├── agent_config.yaml       Skill definitions (YAML)
├── prompts/                One .md per skill (12 files)
├── mcp_runner.py           Multi-turn tool-use loop
├── mcp_server.py           MCP tools (web_search, fetch_url, run_command, etc.)
├── persistence.py          Atomic graph/node state persistence
├── recovery.py             Failure classification + recovery decisions
├── sandbox.py              Isolated subprocess for coder output
├── cache.py                Tool result caching (TTL-based)
├── tracing.py              Per-span JSONL structured logging
├── schemas_v2.py           AgentResult, NodeSpec, NodeState, RunBudget
├── replay.py               Interactive session replay tool
├── dashboard_server.py     FastAPI WebSocket dashboard server
├── dashboard_s8.html       Dashboard frontend (SVG graph + live logs)
├── tests/
│   └── test_recovery.py    26 unit tests (classification + critic splice)
└── agent/                  Session 7 modules (memory, perception, decision, action, gateway)
```

---
