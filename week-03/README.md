# Week 3 — Object Model, Types & Trees

[← Curriculum home](../README.md) · [← Week 2](../week-02/README.md)

**Theme:** This week covers how classes come into existence (and how frameworks hook into that), what a "record" type really costs in memory and CPU, and how a static type system describes Python code without changing how it runs. On the DSA side it's trees: depth-first traversal, breadth-first traversal, and the ordering invariant that makes a binary search tree useful.

The Python days connect. Day 15's class-creation pipeline is exactly how `@dataclass` (Day 17) and Pydantic's `BaseModel` generate code, and Day 19's type checker is what tells you whether that generated code is being used correctly. Day 10's descriptors (Week 2) come back in `__set_name__`.

| Day | Track | Topic | Problem |
|---|---|---|---|
| [15](day-15-metaclasses-class-creation.md) | Python | Metaclasses & the class-creation pipeline | — |
| [16](day-16-tree-dfs.md) | DSA | Tree DFS (return value + global update) | LC 543 · Diameter of Binary Tree |
| [17](day-17-dataclasses-slots-pydantic.md) | Python | Dataclasses, `__slots__` & Pydantic performance | — |
| [18](day-18-tree-bfs.md) | DSA | Tree BFS (level order) | LC 199 · Binary Tree Right Side View |
| [19](day-19-typing-in-depth.md) | Python | Typing in depth: Protocols, generics, variance, ParamSpec | — |
| [20](day-20-bst.md) | DSA | BST invariants (bounds propagation) | LC 98 · Validate Binary Search Tree |
| [21](day-21-consolidation.md) | Audit | Consolidation & self-audit | LC 230 · Kth Smallest Element in a BST |

## Exit criteria for Week 3

By Day 21 you should be able to do the following **without notes**:

- [ ] List the hooks that run when a `class` statement executes, in order, and say which one you'd use for a plugin registry.
- [ ] Explain a "metaclass conflict" error and fix it two different ways.
- [ ] Say what `@dataclass` generates, what `slots=True` and `frozen=True` change, and when a default value is evaluated.
- [ ] Choose between a dataclass, a `NamedTuple`, a `TypedDict` and a Pydantic model for a given boundary, and justify the choice by cost.
- [ ] Write a type-safe decorator with `ParamSpec`, and explain why `list[Dog]` isn't a `list[Animal]`.
- [ ] Write recursive and iterative tree traversals, and convert recursion into an explicit stack when depth is unbounded.
- [ ] Explain why checking only a node's children doesn't validate a BST.

---

[← Week 2](../week-02/README.md) · [Next: Week 4 →](../week-04/README.md)
