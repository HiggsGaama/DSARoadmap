# Week 4 — Concurrency Control, Performance & Graphs I

[← Curriculum home](../README.md) · [← Week 3](../week-03/README.md)

**Theme:** Weeks 1–3 covered how the runtime works. Week 4 is about **keeping a service healthy under real load**: stopping work correctly (cancellation, structured concurrency, graceful shutdown), refusing work you can't handle (bounded queues, load shedding), doing more per unit of hardware (micro-batching for inference), and finding out where the time actually goes (profiling). On the DSA side you add heaps, intervals, and your first graph traversals on grids.

Connections back: cancellation is Day 5's `Task.__step` throwing into a suspended coroutine (Day 3's `throw()`). Micro-batching is Day 5's futures used as a hand-off. Profiling finds Day 1 and Day 2 bugs (list membership, cache misuse) in real code. Heaps come back in Week 5's Dijkstra, and grid BFS is the warm-up for general graphs.

| Day | Track | Topic | Problem |
|---|---|---|---|
| [22](day-22-cancellation-structured-concurrency.md) | Python | Cancellation semantics, `timeout`, `shield`, `TaskGroup`, graceful SIGTERM shutdown | — |
| [23](day-23-heaps-top-k.md) | DSA | Heaps / Top-K (size-k min-heap, quickselect) | LC 215 · Kth Largest Element in an Array |
| [24](day-24-backpressure-micro-batching.md) | Python | Bounded queues, load shedding, dynamic micro-batching for inference | — |
| [25](day-25-intervals.md) | DSA | Intervals (sort + merge, greedy by end, sweep line) | LC 56 · Merge Intervals |
| [26](day-26-profiling.md) | Python | Profiling: cProfile/pstats, py-spy, memray, honest benchmarking | — |
| [27](day-27-graph-grid-traversal.md) | DSA | Grid BFS/DFS, connected components | LC 200 · Number of Islands |
| [28](day-28-consolidation.md) | Audit | Consolidation & self-audit | LC 994 · Rotting Oranges |

## Exit criteria for Week 4

By Day 28 you should be able to do the following **without notes**:

- [ ] Explain where `CancelledError` is delivered, why it's a `BaseException`, and how `asyncio.timeout()` uses `cancelling()`/`uncancel()` to produce `TimeoutError`.
- [ ] Spot a swallowed cancellation in code review and explain the production symptoms (silent timeouts, hung shutdowns).
- [ ] Write a graceful SIGTERM shutdown: stop accepting work, drain in-flight requests under a deadline, then cancel the rest.
- [ ] Size a queue with Little's law and choose between blocking (backpressure) and rejecting (load shedding).
- [ ] Implement a dynamic micro-batcher with per-request futures, `max_batch_size` + `max_wait`, and correct handling of cancelled futures.
- [ ] Choose between cProfile, py-spy and memray for a given symptom, and read a flame graph.
- [ ] Use a size-k min-heap for top-k, merge intervals with the `max(end)` rule, and pick BFS over DFS for shortest paths on grids.

---

[Next: Week 5 →](../week-05/README.md)
