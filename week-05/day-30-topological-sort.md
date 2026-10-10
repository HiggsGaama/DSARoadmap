# Day 30 — Topological Sort: Ordering a Dependency Graph (and Detecting Cycles)

[← Week 5](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 10 · attempt 20 · walkthrough 20 · variants 5)

> **Platform tie-in:** Topological sort is what `pip`/`uv` dependency resolution, Airflow/Dagster DAG scheduling, Terraform's apply order, Makefiles, and Python's own `graphlib.TopologicalSorter` all do. Yesterday's circular import is a cycle in the module graph.

---

## 1. Algorithmic Pattern

A **topological order** of a directed graph lists every node so that for every edge `u → v`, `u` comes before `v`. It exists **if and only if the graph is acyclic** (a DAG). Every topo-sort algorithm is therefore also a cycle detector.

### Variant A: Kahn's algorithm (BFS on in-degrees)

```text
indegree[v] = number of edges into v
queue = all nodes with indegree 0            # no unmet prerequisites
while queue:
    u = queue.popleft(); order.append(u)
    for v in graph[u]:
        indegree[v] -= 1                     # u is done, so one prerequisite of v is satisfied
        if indegree[v] == 0: queue.append(v)
len(order) < n  →  cycle                     # nodes on a cycle never reach indegree 0
```

**Why the cycle check works:** every node on a cycle has an incoming edge from another node on the same cycle. None of them can be the first to reach in-degree 0, so none is ever dequeued.

### Variant B: DFS with three colours

- **WHITE**: unvisited. **GRAY**: on the current DFS path (recursion stack). **BLACK**: fully explored, along with all its descendants.
- Reaching a **GRAY** node means a **back edge**, which means a cycle.
- Appending a node to a list when it turns BLACK (post-order) and **reversing** that list gives a topological order.

Two colours (visited / not visited) **aren't enough** for directed graphs. In the diamond `A→B, A→C, B→D, C→D`, D is reached twice without any cycle. GRAY distinguishes "on my current path" from "finished earlier".

| | Kahn (BFS) | DFS three-colour |
|---|---|---|
| Output | Order emerges directly; naturally gives "levels" (parallelizable batches) | Reversed post-order |
| Cycle report | Count check (`len(order) < n`) | Immediate on the first back edge; the GRAY stack *is* the cycle path |
| Python risk | None | Recursive version hits `RecursionError` at depth ~1000. Use an explicit stack |

Both run in $O(V + E)$ time and space.

## 2. Identification Rule

Use topological sort when:

1. ✅ The problem has **prerequisites, dependencies, "must come before", build order, task scheduling, or install order**.
2. ✅ The input is a list of **directed pairs** `(a, b)` that mean an ordering constraint.
3. ✅ The question is **"is it possible?"** (cycle detection), **"give a valid order"**, or **"minimum number of rounds/semesters"** (Kahn level count).
4. ✅ You need to derive an order from **pairwise comparisons**, e.g. Alien Dictionary (letter order from sorted words).

**Anti-signals:**

- ❌ **Undirected** graph + "is there a cycle / are these connected?" → Union-Find (Day 32) or plain DFS.
- ❌ **Weighted shortest path** → Dijkstra (Day 34). (Shortest path *on a DAG* is a topo sort + relaxation in $O(V + E)$, a useful exception.)

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Course Schedule` | **NeetCode** | DFS cycle detection with a visiting set, drawn out clearly |
| `take U forward topological sort Kahn's algorithm` | **take U forward** (Striver) | Graph series: Kahn's BFS, DFS topo, and cycle detection in directed graphs as separate lessons |
| `William Fiset topological sort` | **WilliamFiset** | Animated, algorithm-first explanation |

---

## 4. Daily Problem — LeetCode 207: Course Schedule (Medium)

**Statement:** There are `numCourses` courses labelled `0..numCourses−1`. `prerequisites[i] = [a, b]` means you must take course `b` **before** course `a`. Return `True` if you can finish all courses, otherwise `False`.

```text
Input:  numCourses = 2, prerequisites = [[1, 0]]
Output: True            # take 0, then 1

Input:  numCourses = 2, prerequisites = [[1, 0], [0, 1]]
Output: False           # 0 needs 1 and 1 needs 0: a cycle

Input:  numCourses = 4, prerequisites = [[1, 0], [2, 0], [3, 1], [3, 2]]
Output: True            # 0 -> {1, 2} -> 3
```

**Constraints:** `1 <= numCourses <= 2000`, `0 <= len(prerequisites) <= 5000`, all pairs unique.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) Which direction should the edge `[a, b]` point? (2) Which courses can you take on day one? (3) What's true about courses you can *never* reach?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. Model courses as nodes, with an edge `b → a` ("b unlocks a"). Getting this direction wrong is the most common bug, although for a pure feasibility check either direction gives the same answer, because reversing every edge preserves cycles.
2. "Can I finish everything?" is the same as "does this directed graph have no cycle?"
3. **Kahn:** start with every course that has no prerequisites. Taking a course removes its outgoing edges, which may free up new courses. If you get stuck before taking all courses, the remaining ones depend on each other circularly.
4. Count how many courses you took. Comparing that count with `numCourses` is the entire answer.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(V + E)$. Each course is enqueued once, and each edge is relaxed once.
- **Space:** $O(V + E)$ for the adjacency list, in-degrees and queue.

### Code: Kahn's algorithm (primary)

```python
from __future__ import annotations

from collections import deque


class Solution:
    def canFinish(self, numCourses: int, prerequisites: list[list[int]]) -> bool:
        graph: list[list[int]] = [[] for _ in range(numCourses)]
        indegree = [0] * numCourses
        for course, prereq in prerequisites:
            graph[prereq].append(course)      # prereq unlocks course
            indegree[course] += 1

        queue = deque(c for c in range(numCourses) if indegree[c] == 0)
        taken = 0
        while queue:
            current = queue.popleft()
            taken += 1
            for nxt in graph[current]:
                indegree[nxt] -= 1            # one fewer unmet prerequisite
                if indegree[nxt] == 0:
                    queue.append(nxt)

        # Courses on (or downstream of) a cycle never reach indegree 0.
        return taken == numCourses


def can_finish_dfs(num_courses: int, prerequisites: list[list[int]]) -> bool:
    """Three-colour DFS with an EXPLICIT stack: a 2000-course chain would blow
    Python's default recursion limit (~1000) with the recursive version."""
    graph: list[list[int]] = [[] for _ in range(num_courses)]
    for course, prereq in prerequisites:
        graph[prereq].append(course)

    WHITE, GRAY, BLACK = 0, 1, 2
    color = [WHITE] * num_courses
    for start in range(num_courses):
        if color[start] != WHITE:
            continue
        color[start] = GRAY
        stack: list[tuple[int, int]] = [(start, 0)]       # (node, index of next neighbour)
        while stack:
            node, i = stack[-1]
            if i < len(graph[node]):
                stack[-1] = (node, i + 1)
                nxt = graph[node][i]
                if color[nxt] == GRAY:                    # back edge -> cycle
                    return False
                if color[nxt] == WHITE:
                    color[nxt] = GRAY
                    stack.append((nxt, 0))
            else:
                color[node] = BLACK                       # all descendants done
                stack.pop()
    return True


if __name__ == "__main__":
    s = Solution()
    cases: list[tuple[int, list[list[int]], bool]] = [
        (2, [[1, 0]], True),
        (2, [[1, 0], [0, 1]], False),
        (4, [[1, 0], [2, 0], [3, 1], [3, 2]], True),       # diamond: no cycle
        (1, [], True),
        (3, [[0, 1], [1, 2], [2, 0]], False),              # 3-cycle
        (5, [[1, 0], [2, 1], [3, 2], [1, 3]], False),      # cycle not containing node 0
        (2000, [[i + 1, i] for i in range(1999)], True),   # deep chain
    ]
    for n, prereqs, expected in cases:
        assert s.canFinish(n, prereqs) is expected, (n, prereqs)
        assert can_finish_dfs(n, prereqs) is expected, (n, prereqs)
    print("all tests passed")
```

### Dry run (Kahn): `n = 4`, `[[1,0],[2,0],[3,1],[3,2]]`

| Step | Dequeue | In-degrees after `[0,1,2,3]` | Queue | Taken |
|---|---|---|---|---|
| init | — | `[0, 1, 1, 2]` | `[0]` | 0 |
| 1 | 0 | `[0, 0, 0, 2]` | `[1, 2]` | 1 |
| 2 | 1 | `[0, 0, 0, 1]` | `[2]` | 2 |
| 3 | 2 | `[0, 0, 0, 0]` | `[3]` | 3 |
| 4 | 3 | — | `[]` | 4 = n ✓ |

### Common mistakes

- **Two-colour DFS** (a single `visited` set) reports a false cycle on the diamond.
- **Recursive DFS** in Python on a long chain gives `RecursionError` at around depth 1000. The 2000-chain test above catches it.
- Building the graph as `graph[course].append(prereq)` and then using Kahn's in-degrees from the other direction. Pick one direction and keep it consistent.

### Variants to try (optional)

- **LC 210 Course Schedule II:** return the order itself (Kahn's `order` list, or reversed DFS post-order).
- **LC 1136 Parallel Courses:** minimum semesters = number of Kahn *levels* (process the queue level by level).
- **LC 269 Alien Dictionary:** build edges from the first differing letter of adjacent words. Watch for the prefix edge case (`"abc"` before `"ab"` is invalid).
- **Stdlib:** `graphlib.TopologicalSorter` (3.9+) has `static_order()` and a `prepare()`/`get_ready()`/`done()` API for running independent tasks in parallel, and raises `CycleError` naming the cycle.

</details>

---

[← Day 29 — Import System](day-29-import-system-packaging.md) · [Day 31 — ASGI & FastAPI Internals →](day-31-asgi-fastapi-internals.md)
