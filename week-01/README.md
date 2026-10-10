# Week 1 — Runtime Foundations & Core Array Patterns

[← Curriculum home](../README.md)

**Theme:** This week covers what the interpreter does with your objects, how execution gets suspended and resumed (generators, then asyncio), and the three array/string patterns behind a large share of interview and production string-processing problems.

The Python days build on each other. Day 3's suspended generator frames are what Day 5's `async`/`await` is built from, and Day 1's reference counting explains why both generators and tasks can silently outlive or die before the code you expect.

| Day | Track | Topic | Problem |
|---|---|---|---|
| [1](day-01-cpython-memory-model.md) | Python | CPython memory model: refcounting, cyclic GC, pymalloc | — |
| [2](day-02-hashing.md) | DSA | Hashing for O(1) lookups | LC 128 · Longest Consecutive Sequence |
| [3](day-03-generators.md) | Python | Generators as suspended-frame state machines | — |
| [4](day-04-two-pointers.md) | DSA | Two Pointers (opposite ends) | LC 15 · 3Sum |
| [5](day-05-asyncio-event-loop.md) | Python | asyncio event loop internals | — |
| [6](day-06-sliding-window.md) | DSA | Sliding Window (variable size) | LC 3 · Longest Substring Without Repeating Characters |
| [7](day-07-consolidation.md) | Audit | Consolidation & self-audit | LC 11 · Container With Most Water |

## Exit criteria for Week 1

By Day 7 you should be able to do the following **without notes**:

- [ ] Explain why `del x` doesn't necessarily free memory, and why freed memory doesn't necessarily lower RSS.
- [ ] Use `tracemalloc` snapshot diffs to tell a real leak apart from allocator fragmentation.
- [ ] Draw the states of a generator and say what `send`, `throw` and `close` do in each state.
- [ ] Explain how a `Task` drives a coroutine and what "the loop is blocked" means mechanically.
- [ ] Recognize hashing, two-pointer and sliding-window problems from the statement alone, and say why the sliding-window invariant holds.

---

[Next: Week 2 →](../week-02/README.md)
