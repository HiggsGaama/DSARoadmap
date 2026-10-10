# Backend & AI Platform Engineering — 6-Week Plan

A day-by-day curriculum for an infrastructure engineer moving into backend and AI platform work. Each session fits in **45–60 minutes** and is aimed at the failure modes you hit in production.

## Weekly rhythm

| Day type | When | Focus | Format |
|---|---|---|---|
| **Python Systems** | Mon / Wed / Fri (odd days) | How CPython and its runtime actually work | Core concept → production gotcha → runnable code → video |
| **DSA Mastery** | Tue / Thu / Sat (even days) | One algorithmic pattern per day | Pattern → identification rule → video → problem → full walkthrough |
| **Consolidation** | Day 7, 14, 21, 28, 35, 42 | Retrieval, transfer, self-audit | Quiz → pattern drill → code review → timed problem |

## How to use a session

1. **Python days:** read the core concept (15 min), then type out and run the code yourself rather than copy-pasting it (20 min). Break it on purpose. Watch the video only if something is still unclear.
2. **DSA days:** read the pattern and identification rule (10 min), then **attempt the problem for 20 minutes before opening the solution**. If you get stuck, open the *Intuition* section only, then try again.
3. **Consolidation days:** answer each question without looking first. Every miss goes into your gap log.

## Setup

```bash
python3 --version          # 3.12+ required (PEP 695 generics, immortal objects, TaskGroup)
python3 -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install mypy ruff       # optional: type-check and lint the examples
```

Every code block is self-contained: save it as a `.py` file and run it. DSA solutions include `assert`-based tests.

## Curriculum

### [Week 1 — Runtime Foundations & Core Array Patterns](week-01/README.md)

| Day | Track | Topic |
|---|---|---|
| [Day 1](week-01/day-01-cpython-memory-model.md) | Python | CPython memory model: refcounting, cyclic GC, pymalloc |
| [Day 2](week-01/day-02-hashing.md) | DSA | Hashing for O(1) lookups — *Longest Consecutive Sequence* |
| [Day 3](week-01/day-03-generators.md) | Python | Generators as suspended-frame state machines |
| [Day 4](week-01/day-04-two-pointers.md) | DSA | Two Pointers — *3Sum* |
| [Day 5](week-01/day-05-asyncio-event-loop.md) | Python | asyncio event loop internals |
| [Day 6](week-01/day-06-sliding-window.md) | DSA | Sliding Window — *Longest Substring Without Repeating Characters* |
| [Day 7](week-01/day-07-consolidation.md) | Audit | Consolidation & self-audit |

### [Week 2 — Runtime Mechanics & Linear Structures](week-02/README.md)

| Day | Track | Topic |
|---|---|---|
| [Day 8](week-02/day-08-context-managers-exceptions.md) | Python | Context managers & exception internals |
| [Day 9](week-02/day-09-monotonic-stack.md) | DSA | Monotonic Stack — *Daily Temperatures* |
| [Day 10](week-02/day-10-descriptors-attribute-lookup.md) | Python | Descriptors, attribute lookup & MRO |
| [Day 11](week-02/day-11-binary-search.md) | DSA | Binary Search on the answer — *Koko Eating Bananas* |
| [Day 12](week-02/day-12-gil-threads-processes.md) | Python | GIL, threads, processes & free-threading |
| [Day 13](week-02/day-13-linked-lists-fast-slow.md) | DSA | Fast/Slow Pointers — *Linked List Cycle II* |
| [Day 14](week-02/day-14-consolidation.md) | Audit | Consolidation & self-audit |

### [Week 3 — Object Model, Types & Trees](week-03/README.md)

| Day | Track | Topic |
|---|---|---|
| [Day 15](week-03/day-15-metaclasses-class-creation.md) | Python | Metaclasses & the class-creation pipeline |
| [Day 16](week-03/day-16-tree-dfs.md) | DSA | Tree DFS — *Diameter of Binary Tree* |
| [Day 17](week-03/day-17-dataclasses-slots-pydantic.md) | Python | Dataclasses, `__slots__` & Pydantic performance |
| [Day 18](week-03/day-18-tree-bfs.md) | DSA | Tree BFS — *Binary Tree Right Side View* |
| [Day 19](week-03/day-19-typing-in-depth.md) | Python | Typing in depth: Protocols, generics, variance |
| [Day 20](week-03/day-20-bst.md) | DSA | BST invariants — *Validate BST* |
| [Day 21](week-03/day-21-consolidation.md) | Audit | Consolidation & self-audit |

### [Week 4 — Concurrency Control, Performance & Graphs I](week-04/README.md)

| Day | Track | Topic |
|---|---|---|
| [Day 22](week-04/day-22-cancellation-structured-concurrency.md) | Python | Cancellation & structured concurrency |
| [Day 23](week-04/day-23-heaps-top-k.md) | DSA | Heaps / Top-K — *Kth Largest Element* |
| [Day 24](week-04/day-24-backpressure-micro-batching.md) | Python | Backpressure & micro-batching for inference |
| [Day 25](week-04/day-25-intervals.md) | DSA | Intervals — *Merge Intervals* |
| [Day 26](week-04/day-26-profiling.md) | Python | Profiling: cProfile, py-spy, memray |
| [Day 27](week-04/day-27-graph-grid-traversal.md) | DSA | Grid BFS/DFS — *Number of Islands* |
| [Day 28](week-04/day-28-consolidation.md) | Audit | Consolidation & self-audit |

### [Week 5 — Service Plumbing & Graphs II](week-05/README.md)

| Day | Track | Topic |
|---|---|---|
| [Day 29](week-05/day-29-import-system-packaging.md) | Python | Import system & packaging |
| [Day 30](week-05/day-30-topological-sort.md) | DSA | Topological Sort — *Course Schedule* |
| [Day 31](week-05/day-31-asgi-fastapi-internals.md) | Python | ASGI & FastAPI internals |
| [Day 32](week-05/day-32-union-find.md) | DSA | Union-Find — *Redundant Connection* |
| [Day 33](week-05/day-33-serialization-performance.md) | Python | Serialization performance & safety |
| [Day 34](week-05/day-34-dijkstra.md) | DSA | Dijkstra — *Network Delay Time* |
| [Day 35](week-05/day-35-consolidation.md) | Audit | Consolidation & self-audit |

### [Week 6 — Production AI Services & Dynamic Programming](week-06/README.md)

| Day | Track | Topic |
|---|---|---|
| [Day 36](week-06/day-36-observability-contextvars-otel.md) | Python | Observability: contextvars, structured logs, OpenTelemetry |
| [Day 37](week-06/day-37-dp-1d.md) | DSA | 1-D DP — *Coin Change* |
| [Day 38](week-06/day-38-resilience-patterns.md) | Python | Resilience: retries, circuit breakers, rate limiting |
| [Day 39](week-06/day-39-dp-2d.md) | DSA | 2-D DP — *Longest Common Subsequence* |
| [Day 40](week-06/day-40-streaming-llm-responses.md) | Python | Streaming LLM responses (SSE + async generators) |
| [Day 41](week-06/day-41-backtracking-tries.md) | DSA | Backtracking & Tries — *Combination Sum* |
| [Day 42](week-06/day-42-final-consolidation.md) | Audit | Final capstone consolidation |

## Gap log template

Keep a `GAPS.md` next to your code. One line per miss:

```text
| Date | Day | What I got wrong | Correct model (1 sentence) | Re-test on |
```
