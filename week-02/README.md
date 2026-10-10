# Week 2 — Resource Safety, Attribute Machinery & Concurrency Primitives

[← Curriculum home](../README.md) · [← Week 1](../week-01/README.md)

**Theme:** Week 1 covered how objects live, die and get suspended. Week 2 covers what happens *around* your code: how `with` guarantees cleanup (and how it quietly stops guaranteeing it), how `obj.attr` actually resolves, and what the GIL really prevents and what it doesn't. On the DSA side you add three patterns that come up constantly: monotonic stacks, binary search over an answer space, and fast/slow pointers.

There are two connections back to Week 1. `@contextmanager` is a generator (Day 3) driven by `throw()`. Bound methods, `property` and `cached_property` are all descriptors, which explains several Day 1 leaks and caching bugs.

| Day | Track | Topic | Problem |
|---|---|---|---|
| [8](day-08-context-managers-exceptions.md) | Python | Context managers, exception chaining, `ExceptionGroup` / `except*` | — |
| [9](day-09-monotonic-stack.md) | DSA | Stack / Monotonic Stack | LC 739 · Daily Temperatures |
| [10](day-10-descriptors-attribute-lookup.md) | Python | Descriptors, attribute lookup order, MRO & `super()` | — |
| [11](day-11-binary-search.md) | DSA | Binary Search & search-on-answer | LC 875 · Koko Eating Bananas |
| [12](day-12-gil-threads-processes.md) | Python | The GIL, threads vs processes vs asyncio, fork vs spawn | — |
| [13](day-13-linked-lists-fast-slow.md) | DSA | Fast / Slow Pointers (Floyd) | LC 142 · Linked List Cycle II |
| [14](day-14-consolidation.md) | Audit | Consolidation & self-audit | LC 153 · Find Minimum in Rotated Sorted Array |

## Exit criteria for Week 2

By Day 14 you should be able to do the following **without notes**:

- [ ] Write out what `with` desugars to, and say exactly when `__exit__` suppresses an exception.
- [ ] Explain why a `@contextmanager` without `try/finally` around `yield` leaks resources on error.
- [ ] Recite the attribute lookup order (data descriptor → instance `__dict__` → non-data descriptor → class attribute → `__getattr__`).
- [ ] Explain why `obj.method is obj.method` is `False`, and why `cached_property` can be shadowed and goes stale.
- [ ] Say when threads give real parallelism in CPython, and why `fork()` in a process that already has threads can deadlock.
- [ ] Write the "first True of a monotone predicate" binary search template from memory, with no off-by-one errors.
- [ ] Prove why Floyd's second phase lands on the cycle entrance.
