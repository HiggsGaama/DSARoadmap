# Week 5 — Service Plumbing & Graphs II

[← Curriculum home](../README.md)

**Theme:** This week covers the plumbing between your code and production: how modules get loaded (and why that crashes at boot), how a request travels through an ASGI server into FastAPI, and how data crosses the wire without losing types or opening security holes. The DSA track finishes the core graph algorithms: ordering (topological sort), connectivity (Union-Find), and weighted shortest paths (Dijkstra).

The Python days build on each other. Day 29's rule "no I/O at import time" leads into Day 31's **lifespan** handler, and Day 31's response pipeline is where Day 33's serialization costs show up. On the DSA side, Day 30's cycle detection, Day 32's connectivity and Day 34's greedy relaxation all reappear in Day 35's transfer problem, which can be solved three different ways.

| Day | Track | Topic | Problem |
|---|---|---|---|
| [29](day-29-import-system-packaging.md) | Python | Import system & packaging: finders, loaders, cycles, wheels | — |
| [30](day-30-topological-sort.md) | DSA | Topological Sort (Kahn + three-colour DFS) | LC 207 · Course Schedule |
| [31](day-31-asgi-fastapi-internals.md) | Python | ASGI & FastAPI internals: scope/receive/send, middleware, lifespan, threadpool | — |
| [32](day-32-union-find.md) | DSA | Union-Find (path compression + union by size) | LC 684 · Redundant Connection |
| [33](day-33-serialization-performance.md) | Python | Serialization: speed, size, silent type loss, pickle security | — |
| [34](day-34-dijkstra.md) | DSA | Dijkstra with `heapq` + lazy deletion | LC 743 · Network Delay Time |
| [35](day-35-consolidation.md) | Audit | Consolidation & self-audit (+ spaced repetition from Weeks 1–4) | LC 1631 · Path With Minimum Effort |

## Exit criteria for Week 5

By Day 35 you should be able to do the following **without notes**:

- [ ] Trace `import a.b` through `sys.modules`, `sys.meta_path`, specs and loaders, and explain why a circular `from x import y` fails while `import x` often doesn't.
- [ ] Explain why I/O at import time is dangerous with forking servers, and where that work belongs instead (lifespan / explicit startup).
- [ ] Write a raw ASGI app and a streaming-safe pure-ASGI middleware, and explain the sync-endpoint threadpool limit.
- [ ] Pick serialization formats for an API, a cache and an internal RPC; list the types JSON silently loses; explain why `pickle.loads` on untrusted bytes is RCE.
- [ ] Implement Kahn's algorithm, three-colour DFS, Union-Find (with both optimizations) and lazy-deletion Dijkstra from memory, and say when each one *doesn't* apply.

---

[← Week 4](../week-04/README.md) · [Next: Week 6 →](../week-06/README.md)
