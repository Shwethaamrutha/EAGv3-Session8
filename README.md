# ΛXON

### Multi-Agent DAG Orchestrator

> *From a single iterating loop to a directed acyclic graph of specialised skills. The agent plans the graph; the executor runs it; nodes that share no dependencies run in parallel.*

[![Python](https://img.shields.io/badge/Python-3.11+-blue)](https://python.org)
[![NetworkX](https://img.shields.io/badge/Graph-NetworkX-green)](https://networkx.org)
[![Bedrock](https://img.shields.io/badge/LLM-AWS%20Bedrock-purple)](https://aws.amazon.com/bedrock/)
[![FAISS](https://img.shields.io/badge/Memory-FAISS%20768d-orange)](https://github.com/facebookresearch/faiss)

---

**ΛXON** is a DAG-based multi-agent orchestrator built as Session 8 of the [EAG V3](https://theschoolof.ai) program. It extends the Session 7 single-loop agent (Memory → Perception → Decision → Action) into a growing directed acyclic graph where a Planner decomposes queries into skill nodes, an Executor runs them concurrently via `asyncio.gather`, and a Critic verifies outputs with tool-grounded measurements.

**What makes it different from Session 7:**
- Parallel execution — three independent sub-tasks run concurrently, not sequentially
- Explicit graph structure — the trace is a diagram, not a list of iterations
- Per-skill scoping — each node sees only its relevant inputs, not cumulative history
- Tool-grounded critic — verification uses actual measurement tools, not LLM judgment
- Resumable execution — kill at any point, resume from the last completed node

---

## Architecture

```
USER QUERY
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PLANNER (LLM)                                                       │
│  Reads: query + FAISS memory hits                                    │
│  Emits: JSON graph of skill nodes with typed inputs/outputs          │
└────────────────────────┬────────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
    ┌───────────┐  ┌───────────┐  ┌───────────┐
    │ researcher│  │ researcher│  │ researcher│    ← parallel via asyncio.gather
    │ (London)  │  │ (Paris)   │  │ (Berlin)  │
    └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   comparator    │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │    formatter    │  → FINAL ANSWER
                └─────────────────┘
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

## Performance Comparison

| Query | Session 7 (single-loop) | Session 8 (DAG) | Speedup |
|-------|------------------------|-----------------|---------|
| Shannon (fetch + extract) | 24s, 8 iterations | 11s, 3 nodes | 2.2x |
| Populations (3-city compare) | 125s, 11 iterations | 25-35s, 6 nodes | 3.5-5x |
| Hello (trivial) | 8s, 2 iterations | 7s, 2 nodes | 1.1x |
| Graceful failure | 8s, 3 iterations | 3s, 2 nodes | 2.7x |

**Why Session 8 is faster:**
- Parallel fan-out: 3 researchers run concurrently (wall-clock = max branch, not sum)
- No cumulative history: each node sees only its inputs, not 10 iterations of context
- Direct tool dispatch: researchers skip LLM mediation for factual lookups

**Why Session 7 wasn't wrong:**
- Sequential queries (Shannon) show minimal speedup — DAG overhead ~= loop overhead
- The DAG wins only when there's real parallelism to exploit

---

## Token Efficiency

| Architecture | Populations Query | Why |
|---|---|---|
| Session 7 | ~54,000 input tokens | History re-sent every iteration (O(n^2) in iterations) |
| Session 8 | ~17,000 input tokens | Each node scoped to its inputs only |

The 3x token reduction comes from eliminating the quadratic history growth. Session 7 sends the entire run history to Perception and Decision on every iteration. Session 8 nodes see only their typed inputs.

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

## Running

```bash
# Start dashboard
python dashboard_server.py
# Open http://localhost:8080

# CLI mode
python flow.py "Find the populations of London, Paris, Berlin and tell me which two are closest in size"

# Resume a killed session
python flow.py --resume s8_XXXXXXXX

# Run tests
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

## Honest Limitations

1. **Syllable counting** — the critic tool uses a rule-based counter. Not 100% accurate for edge cases (silent-e, compound words). Session 9 forward pointer.

2. **Character-count tasks** — the coder can't reliably write a string of exact length. The model doesn't count while generating. Architecture correctly detects failure via critic, but recovery often fails too.

3. **Mid-tool-call resume** — a researcher killed 3 hops deep into its tool loop restarts from scratch on resume. Only node-boundary persistence.

4. **LLM compliance** — models sometimes return markdown instead of JSON, ignore "no assert" instructions, or hallucinate tools. Contract validation catches this but costs a retry.


---

## Assignment Checklist

| # | Requirement | Status |
|---|-------------|--------|
| 1 | Pass base queries (hello, A, I, J, K) | Done |
| 2 | Parallel fan-out query (3+ concurrent nodes) | Done — populations query, 3.85x speedup |
| 3 | Critic pass + fail across runs | Done — tweet 140-char (FAIL) + 3 sentences (PASS) |
| 4 | Coder skill with computation | Done — compound interest with sandbox verification |
| 5 | New skill (shell + fact_checker) | Done — YAML + prompt, no orchestrator change |
| 6 | YouTube demo | [Link TBD] |
| 7 | README with logs | This file |
