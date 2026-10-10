# Day 34 — Dijkstra: Shortest Paths with a Heap and Lazy Deletion

[← Week 5](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 15 · attempt 20 · walkthrough 15 · variants 5)

> **Platform tie-in:** Weighted shortest paths show up in latency-aware routing (pick the replica path with the lowest summed RTT), service-mesh failover, cost-based query planning, and "cheapest chain of model calls" planners. Day 23's heap is the engine.

---

## 1. Algorithmic Pattern

Single-source shortest paths on a graph with **non-negative edge weights**.

```text
dist[source] = 0; heap = [(0, source)]
while heap:
    d, u = heappop(heap)                 # closest not-yet-finalized node
    if d > dist[u]: continue             # STALE entry: u was already finalized with a smaller d
    for v, w in graph[u]:
        if d + w < dist[v]:              # relax the edge
            dist[v] = d + w
            heappush(heap, (dist[v], v)) # push a NEW entry; don't try to update the old one
```

### Why it's correct: the greedy invariant

When `u` is popped with distance `d` (and the entry isn't stale), `d` is final. Any other path to `u` must leave the finalized set through some node `x` still in the heap with `dist[x] ≥ d`. With **non-negative** weights, that path can't come back cheaper than `d`.

**That's why negative edges break it.** A negative edge found later could reduce the distance of a node that's already finalized. Use **Bellman-Ford** ($O(VE)$, which also detects negative cycles) or, on a DAG, topo-order relaxation (Day 30).

### Lazy deletion (the Python-specific part)

Textbook Dijkstra uses a heap with **decrease-key**, and `heapq` doesn't have one. Instead, push a fresh `(new_dist, v)` entry whenever you improve `v`, and **skip stale entries when they're popped** (`if d > dist[u]: continue`). The heap can hold up to $O(E)$ entries, so the time is $O(E \log E) = O(E \log V)$, since $E \le V^2$ gives $\log E \le 2 \log V$. This is simpler and, in practice, faster than an indexed heap with decrease-key.

### Complexity

- **Time:** $O((V + E) \log V)$ with a binary heap.
- **Space:** $O(V + E)$.

| Graph type | Algorithm |
|---|---|
| Unweighted (all weights equal) | **BFS** (Day 27): $O(V + E)$, no heap |
| Weights 0 or 1 only | **0-1 BFS** with a deque: push weight-0 edges to the front |
| Non-negative weights | **Dijkstra** |
| Negative edges | Bellman-Ford / SPFA |
| All pairs, small V | Floyd-Warshall, $O(V^3)$ |

## 2. Identification Rule

Use Dijkstra when:

1. ✅ There's a **weighted** graph (explicit or implicit: grids with costs, state spaces) and you need the **minimum total cost/time/distance** from a source.
2. ✅ The weights are **non-negative**. Check the constraints.
3. ✅ The phrasing is **"minimum time for a signal to reach all nodes"**, "cheapest path", "least effort", or "probability of the most reliable path" (maximize a product: use max-heap or log-weights).
4. ✅ The path cost is **monotone along a path** even when it isn't a sum, e.g. "minimize the *maximum* edge on the path" (minimax). Dijkstra still works with `max` in place of `+` (Day 35's transfer problem).

**Anti-signals:**

- ❌ All edges cost the same → BFS is simpler and $O(V + E)$.
- ❌ "At most K stops" (LC 787) → the stop constraint breaks the greedy invariant. Use Bellman-Ford bounded to K rounds, or BFS by levels.
- ❌ Negative weights → Bellman-Ford.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Network Delay Time Dijkstra` | **NeetCode** | Today's problem, with the heap-based template |
| `take U forward Dijkstra algorithm priority queue` | **take U forward** (Striver) | Why visited checks or stale skips matter; set vs priority-queue versions |
| `William Fiset Dijkstra shortest path` | **WilliamFiset** | Lazy vs eager (indexed-heap) Dijkstra explained side by side |

---

## 4. Daily Problem — LeetCode 743: Network Delay Time (Medium)

**Statement:** There are `n` nodes labelled `1..n`. `times[i] = [u, v, w]` is a directed edge: a signal from `u` reaches `v` after `w` time units. A signal is sent from node `k`. Return the minimum time for **all** `n` nodes to receive it, or `-1` if some node is unreachable.

```text
Input:  times = [[2,1,1], [2,3,1], [3,4,1]], n = 4, k = 2
Output: 2              # 2->1 (1), 2->3 (1), 2->3->4 (2); the slowest arrival is 2

Input:  times = [[1,2,1]], n = 2, k = 1
Output: 1

Input:  times = [[1,2,1]], n = 2, k = 2
Output: -1             # node 1 is unreachable from 2
```

**Constraints:** `1 <= k <= n <= 100`, `1 <= len(times) <= 6000`, `0 <= w <= 100`, no duplicate edges.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) "Time for all nodes to receive" is which aggregate of the shortest distances? (2) What does your heap pop if a node was pushed twice?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. The signal spreads along every edge at once, so each node receives it at its **shortest-path distance** from `k`.
2. Everyone has it when the **last** node gets it, so the answer is `max(dist)`. If any `dist` is infinite, return `-1`.
3. Weights are non-negative (`0 <= w`), so Dijkstra applies. Weight-0 edges are fine.
4. Use a min-heap of `(distance, node)` and skip stale pops. Optionally, stop early once all `n` nodes are finalized.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(E \log E)$. Each successful relaxation pushes once, giving at most $E$ pushes and pops.
- **Space:** $O(V + E)$ for the adjacency list, `dist` and the heap.

### Code

```python
from __future__ import annotations

import heapq
import math
from collections import defaultdict


class Solution:
    def networkDelayTime(self, times: list[list[int]], n: int, k: int) -> int:
        graph: defaultdict[int, list[tuple[int, int]]] = defaultdict(list)
        for u, v, w in times:
            graph[u].append((v, w))

        dist = [math.inf] * (n + 1)          # 1-indexed; dist[0] unused
        dist[k] = 0
        heap: list[tuple[float, int]] = [(0, k)]
        finalized = 0

        while heap:
            d, u = heapq.heappop(heap)
            if d > dist[u]:                   # stale entry: a shorter path was already processed
                continue
            finalized += 1
            if finalized == n:                # every node is final -> d is the max distance
                return int(d)
            for v, w in graph[u]:
                nd = d + w
                if nd < dist[v]:              # relax
                    dist[v] = nd
                    heapq.heappush(heap, (nd, v))

        return -1                             # heap drained before all n nodes were reached


if __name__ == "__main__":
    s = Solution()
    assert s.networkDelayTime([[2, 1, 1], [2, 3, 1], [3, 4, 1]], 4, 2) == 2
    assert s.networkDelayTime([[1, 2, 1]], 2, 1) == 1
    assert s.networkDelayTime([[1, 2, 1]], 2, 2) == -1
    assert s.networkDelayTime([], 1, 1) == 0                                      # single node
    assert s.networkDelayTime([[1, 2, 5], [1, 3, 1], [3, 2, 1]], 3, 1) == 2      # indirect path wins
    assert s.networkDelayTime([[1, 2, 0], [2, 3, 0]], 3, 1) == 0                  # zero weights
    print("all tests passed")
```

**Why the early return is correct:** nodes are finalized in non-decreasing order of distance, so the `n`-th finalized node has the largest shortest-path distance. Returning `d` at that point equals `max(dist)` without scanning the array.

### Dry run: `[[1,2,5],[1,3,1],[3,2,1]]`, `n = 3`, `k = 1`

| Pop `(d, u)` | Stale? | Relaxations | `dist[1..3]` | Heap after |
|---|---|---|---|---|
| (0, 1) | no | 2 ← 5, 3 ← 1 | `[0, 5, 1]` | `[(1,3), (5,2)]` |
| (1, 3) | no | 2 ← 1+1 = 2 < 5 ✓ | `[0, 2, 1]` | `[(2,2), (5,2)]` |
| (2, 2) | no | — (finalized = 3 = n) | | **return 2** |
| (5, 2) | *would be stale*: 5 > dist[2] = 2 | | | (never reached) |

### Common mistakes

- **Marking nodes visited on push instead of on pop.** That finalizes a distance before a cheaper path has been seen, which is wrong.
- **No stale check.** It's still correct with non-negative weights, but every stale pop re-relaxes all neighbours, and on dense graphs that blows up the runtime.
- Returning `max(dist)` with `dist[0]` (unused, `inf`) still in the list makes every answer `-1`. Slice `dist[1:]`.
- Applying it to graphs with negative weights. It produces silently wrong answers, with no error.

### Variants to try (optional)

- **LC 1514 Path with Maximum Probability:** max-heap on the product (push `-prob`).
- **LC 787 Cheapest Flights Within K Stops:** see the anti-signals. Use Bellman-Ford bounded to K+1 rounds.
- **LC 1631 Path With Minimum Effort:** minimax Dijkstra on a grid. **That's tomorrow's transfer problem**, so don't look it up.

</details>

---

[← Day 33 — Serialization](day-33-serialization-performance.md) · [Day 35 — Consolidation →](day-35-consolidation.md)
