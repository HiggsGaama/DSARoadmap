# Day 32 — Union-Find (Disjoint Set Union): Near-Constant-Time Connectivity

[← Week 5](README.md) · **Track:** DSA · **Time box:** 50 min (pattern 15 · attempt 20 · walkthrough 10 · variants 5)

> **Platform tie-in:** DSU is how you do incremental connectivity: clustering near-duplicate documents for embedding dedup (union pairs above a similarity threshold), grouping alerts into incidents, merging user identities across sources, and Kruskal's minimum spanning tree.

---

## 1. Algorithmic Pattern

Union-Find maintains a partition of elements into disjoint sets under two operations:

- `find(x)`: return the **representative** (root) of `x`'s set.
- `union(a, b)`: merge the two sets. It returns `False` if `a` and `b` were already in the same set, and **that return value is a cycle detector for undirected graphs**.

Each set is a tree of `parent` pointers, and a root points to itself.

### The two optimizations, and why you need both

| Optimization | What it does | Alone gives |
|---|---|---|
| **Union by size/rank** | Attach the *smaller* tree under the larger root | Height ≤ $\log n$, so $O(\log n)$ per op |
| **Path compression** | During `find`, point every visited node (or every other node, "path halving") closer to the root | Amortized $O(\log n)$ |
| **Both** | — | Amortized $O(\alpha(n))$, the **inverse Ackermann function**, which is ≤ 4 for any $n$ that fits in the universe. Effectively constant |

Without either one, a chain of unions like `union(1,2), union(2,3), …` builds a linked list, and `find` costs $O(n)$.

### Template

```python
class DSU:
    def __init__(self, n: int) -> None:
        self.parent = list(range(n))
        self.size = [1] * n
        self.components = n

    def find(self, x: int) -> int:
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]   # path halving: iterative, no recursion limit
            x = self.parent[x]
        return x

    def union(self, a: int, b: int) -> bool:
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False                                    # already connected
        if self.size[ra] < self.size[rb]:
            ra, rb = rb, ra
        self.parent[rb] = ra                                # smaller under larger
        self.size[ra] += self.size[rb]
        self.components -= 1
        return True
```

Path *halving* (shown) is the iterative, single-pass variant of path compression with the same complexity bound, and it avoids recursion in Python. Full compression needs two passes, or recursion that breaks past depth ~1000.

**Limits:** DSU only **merges**. It can't split sets or delete edges. If edges disappear, you need offline tricks (process in reverse and add) or other structures.

## 2. Identification Rule

Use Union-Find when:

1. ✅ The problem is about **undirected connectivity**: "number of connected components / provinces / groups", "are x and y connected?"
2. ✅ **Edges arrive incrementally** and you need connectivity answers *between* additions (online), or you need the **first edge that creates a cycle**.
3. ✅ It's about **grouping by transitive equivalence**: "accounts sharing an email belong to the same person", "a == b and b == c means a == c" (LC 990 Satisfiability of Equality Equations).
4. ✅ You need a **minimum spanning tree** (Kruskal: sort the edges, union when it doesn't form a cycle).

**Anti-signals:**

- ❌ **Directed** cycle detection or ordering → topological sort (Day 30). DSU ignores direction.
- ❌ You need **paths or distances**, not just connectivity → BFS / Dijkstra.
- ❌ A static graph with a single connectivity query → plain DFS/BFS is just as fast and simpler.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Redundant Connection` | **NeetCode** | Today's problem, solved with union by rank + path compression |
| `take U forward disjoint set union by rank union by size path compression` | **take U forward** (Striver) | Best explanation of *why* both optimizations matter |
| `William Fiset union find` | **WilliamFiset** | Short animated series: kinds of union, path compression, Kruskal |

---

## 4. Daily Problem — LeetCode 684: Redundant Connection (Medium)

**Statement:** A tree with `n` nodes labelled `1..n` had **one extra edge** added. You're given the resulting graph as an edge list `edges` (length `n`). Return an edge that can be removed so the result is a tree. If there are several answers, return the one that occurs **last** in the input.

```text
Input:  edges = [[1, 2], [1, 3], [2, 3]]
Output: [2, 3]

Input:  edges = [[1, 2], [2, 3], [3, 4], [1, 4], [1, 5]]
Output: [1, 4]
```

**Constraints:** `3 <= n <= 1000`, `edges[i] = [a, b]` with `a < b`, no repeated edges, and the graph is connected.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) Adding an edge between two nodes that are *already connected* creates exactly what? (2) Why does processing in input order automatically give the *last* such edge?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. A tree on `n` nodes has `n − 1` edges. We have `n`, so there's exactly one cycle.
2. Add edges one at a time. Before adding `(a, b)`, check whether `a` and `b` are already connected. If they are, this edge **closes the cycle**: it's redundant.
3. **Why it's the last one in the input:** every edge on the cycle is a valid answer. While processing in order, the first time an edge's endpoints are already connected is when the *last cycle edge* is reached, because the earlier cycle edges were still building the path.
4. A DSU answers "already connected?" in near-constant time. DFS per edge would cost $O(n)$ each, $O(n^2)$ overall.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n \cdot \alpha(n))$, effectively $O(n)$. There are `n` unions, each near-constant amortized.
- **Space:** $O(n)$ for `parent` and `size`.

### Code

```python
from __future__ import annotations


class DSU:
    def __init__(self, n: int) -> None:
        self.parent = list(range(n))
        self.size = [1] * n

    def find(self, x: int) -> int:
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]  # path halving
            x = self.parent[x]
        return x

    def union(self, a: int, b: int) -> bool:
        """Merge the sets of a and b. Returns False if they were already connected."""
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False
        if self.size[ra] < self.size[rb]:   # union by size: keep trees shallow
            ra, rb = rb, ra
        self.parent[rb] = ra
        self.size[ra] += self.size[rb]
        return True


class Solution:
    def findRedundantConnection(self, edges: list[list[int]]) -> list[int]:
        dsu = DSU(len(edges) + 1)          # nodes are 1-indexed; index 0 is unused
        for a, b in edges:
            if not dsu.union(a, b):        # already connected -> this edge closes the cycle
                return [a, b]
        raise ValueError("input contained no redundant edge")


if __name__ == "__main__":
    s = Solution()
    assert s.findRedundantConnection([[1, 2], [1, 3], [2, 3]]) == [2, 3]
    assert s.findRedundantConnection([[1, 2], [2, 3], [3, 4], [1, 4], [1, 5]]) == [1, 4]
    assert s.findRedundantConnection([[3, 4], [1, 2], [2, 4], [3, 5], [2, 5]]) == [2, 5]
    # Long chain + closing edge: exercises path halving on a deep tree.
    n = 1000
    chain = [[i, i + 1] for i in range(1, n)] + [[1, n]]
    assert s.findRedundantConnection(chain) == [1, n]

    dsu = DSU(6)
    for a, b in [(0, 1), (2, 3), (1, 3)]:
        dsu.union(a, b)
    assert dsu.find(0) == dsu.find(2) and dsu.find(4) != dsu.find(0)
    print("all tests passed")
```

### Dry run: `[[1,2],[2,3],[3,4],[1,4],[1,5]]`

| Edge | `find(a)`, `find(b)` | Same root? | Action |
|---|---|---|---|
| (1,2) | 1, 2 | no | union → {1,2} |
| (2,3) | 1, 3 | no | union → {1,2,3} |
| (3,4) | 1, 4 | no | union → {1,2,3,4} |
| (1,4) | 1, 1 | **yes** | **return [1, 4]** |

### Common mistakes

- Forgetting the 1-indexed nodes: `DSU(n)` with node `n` gives an `IndexError`.
- Union without rank/size and without compression: correct, but $O(n)$ per `find` on adversarial chains.
- Recursive `find` with full compression on a 10^5-node chain gives a `RecursionError` before compression ever kicks in.

### Variants to try (optional)

- **LC 547 Number of Provinces:** count components (`dsu.components`).
- **LC 721 Accounts Merge:** union account indices that share an email, then group.
- **LC 685 Redundant Connection II:** the directed version. A node may have two parents, so DSU alone isn't enough. It's a classic follow-up.
- **LC 1584 Min Cost to Connect All Points:** Kruskal = sort edges + DSU.

</details>

---

[← Day 31 — ASGI & FastAPI Internals](day-31-asgi-fastapi-internals.md) · [Day 33 — Serialization →](day-33-serialization-performance.md)
