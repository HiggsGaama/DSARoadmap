# Day 27 — Graph Traversal on Grids: BFS, DFS, and Connected Components

[← Week 4](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 10 · attempt 20 · walkthrough 20 · variants 5)

---

## 1. Algorithmic Pattern

A 2-D grid is a graph in disguise. Each cell `(r, c)` is a node, and edges go to its 4 neighbours (or 8 if diagonals count). You never build an adjacency list. Neighbours are computed on the fly:

```python
DIRECTIONS = ((1, 0), (-1, 0), (0, 1), (0, -1))
for dr, dc in DIRECTIONS:
    nr, nc = r + dr, c + dc
    if 0 <= nr < rows and 0 <= nc < cols and passable(nr, nc) and (nr, nc) not in seen: ...
```

### The two traversals

| | **BFS** (queue: `collections.deque`) | **DFS** (stack or recursion) |
|---|---|---|
| Order | Layer by layer, by distance from the source | Goes deep along one path first |
| Gives you | **Shortest path in unweighted graphs**, distance layers, "minimum steps" | Connectivity, components, cycle detection, flood fill, path existence |
| Memory | The frontier, up to about the grid's width × 2 in a grid | The current path, up to **every cell** in a snake-shaped grid |
| Python caveat | — | Recursive DFS hits `RecursionError` at depth ~1000 (default limit) |

For **counting connected components** (today's problem) either works. Scan every cell, and each time you find an unvisited land cell, increment the count and flood-fill its whole component, marking every cell visited.

**Complexity:** each cell is visited once and each edge checked a constant number of times (4 per cell), so the total is $O(R \cdot C)$ time and $O(R \cdot C)$ visited state.

### Visited marking: mark when you *enqueue*, not when you *dequeue*

If you only mark cells visited when popping them, the same cell can be pushed by several neighbours before it's popped. The queue fills with duplicates and the work multiplies (in a dense grid up to 4× the pushes). Worse, in weighted variants or anything with side effects, the cell gets *processed* more than once. **Mark visited at the moment you add a cell to the queue or stack.**

### Mutating the input as the visited set

The common LeetCode trick is to overwrite `"1"` with `"0"` (or `"#"`) to mark cells visited. It saves memory, but **it destroys the caller's data**. In production code, where the grid might be a cached map tile, a shared config or a numpy array someone else reads, keep a separate `seen` structure, or explicitly document and copy. In interviews, state the trade-off out loud.

## 2. Identification Rule

Use grid BFS/DFS when **any** of these appear:

1. ✅ A **2-D matrix** of cells with "connected", "adjacent (4- or 8-directionally)", "region", "island", "flood fill", "enclosed".
2. ✅ **"Count the number of …" groups** → connected components (DFS or BFS, or Union-Find in Week 5).
3. ✅ **"Minimum number of steps / moves / minutes"** on an unweighted grid → **BFS** (DFS doesn't give shortest paths).
4. ✅ **Many sources spreading at once** (rotting, fire, distance to nearest X) → **multi-source BFS**: seed the queue with all sources at distance 0 (Day 28's transfer problem).
5. ✅ Hidden graphs: word ladders, lock combinations, state spaces. If you can generate "next states" from a state, BFS works.

**Anti-signals:** weighted edges → Dijkstra (Week 5). Dependencies or ordering → topological sort (Week 5). Dynamic connectivity with merges → Union-Find.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Number of Islands` | **NeetCode** | BFS solution with clear visited handling |
| `take U forward number of islands graph series` | **take U forward** (Striver) | Part of a full graph series: BFS/DFS on grids, then on general graphs |
| `William Fiset breadth first search grid shortest path` | **WilliamFiset** | Graph theory fundamentals: BFS on grids, with careful complexity reasoning |

---

## 4. Daily Problem — LeetCode 200: Number of Islands (Medium)

**Statement:** Given an `m × n` grid of `"1"`s (land) and `"0"`s (water), return the number of islands. An island is surrounded by water and is formed by connecting adjacent land cells **horizontally or vertically**. Assume all four edges of the grid are surrounded by water.

```text
Input:
grid = [
  ["1","1","1","1","0"],
  ["1","1","0","1","0"],
  ["1","1","0","0","0"],
  ["0","0","0","0","0"]
]
Output: 1

Input:
grid = [
  ["1","1","0","0","0"],
  ["1","1","0","0","0"],
  ["0","0","1","0","0"],
  ["0","0","0","1","1"]
]
Output: 3
```

**Constraints:** `1 <= m, n <= 300`, cells are `"0"` or `"1"`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) When you find a land cell you haven't seen, what should happen to the rest of its island? (2) The grid can be 300 × 300 of all land. What happens to a recursive solution?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. Each island is one **connected component** of land cells, so the answer is the number of components.
2. **Scan** every cell in row-major order. When you hit land that hasn't been visited, you've found a **new island**: increment the count.
3. Immediately **flood-fill** from that cell (BFS or DFS) and mark every reachable land cell visited, so the scan never counts this island again.
4. Every cell is visited at most once by the flood fills and once by the scan, so the total is linear in the grid size.
5. **Recursion trap:** an all-land 300 × 300 grid creates a single DFS path up to 90,000 calls deep. Python's default recursion limit is 1000, so recursive DFS raises `RecursionError`. Raising the limit with `sys.setrecursionlimit` can crash the interpreter with a C-stack overflow instead. Use BFS with a `deque`, or DFS with an explicit stack.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(R \cdot C)$. Each cell is enqueued at most once, and each dequeue checks 4 neighbours.
- **Space:** $O(R \cdot C)$ for `seen` in the worst case (all land). The BFS queue itself holds $O(\min(R, C))$ cells for a flood from a corner, and more in odd shapes.

### Code

```python
from __future__ import annotations

from collections import deque

DIRECTIONS = ((1, 0), (-1, 0), (0, 1), (0, -1))


class Solution:
    def numIslands(self, grid: list[list[str]]) -> int:
        """BFS flood fill per component. Does NOT mutate the input grid."""
        if not grid or not grid[0]:
            return 0
        rows, cols = len(grid), len(grid[0])
        seen = [[False] * cols for _ in range(rows)]   # faster than a set of tuples
        islands = 0

        for r in range(rows):
            for c in range(cols):
                if grid[r][c] != "1" or seen[r][c]:
                    continue
                islands += 1                           # new, unvisited land -> new island
                seen[r][c] = True                      # mark on ENQUEUE
                queue = deque([(r, c)])
                while queue:
                    cr, cc = queue.popleft()
                    for dr, dc in DIRECTIONS:
                        nr, nc = cr + dr, cc + dc
                        if (0 <= nr < rows and 0 <= nc < cols
                                and grid[nr][nc] == "1" and not seen[nr][nc]):
                            seen[nr][nc] = True        # mark before pushing: no duplicates
                            queue.append((nr, nc))
        return islands


def num_islands_recursive(grid: list[list[str]]) -> int:
    """Textbook recursive DFS. Correct, but crashes on large connected regions in Python."""
    rows, cols = len(grid), len(grid[0])
    seen: set[tuple[int, int]] = set()

    def dfs(r: int, c: int) -> None:
        if not (0 <= r < rows and 0 <= c < cols) or grid[r][c] != "1" or (r, c) in seen:
            return
        seen.add((r, c))
        for dr, dc in DIRECTIONS:
            dfs(r + dr, c + dc)

    count = 0
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == "1" and (r, c) not in seen:
                count += 1
                dfs(r, c)
    return count


if __name__ == "__main__":
    s = Solution()
    g1 = [list("11110"), list("11010"), list("11000"), list("00000")]
    g2 = [list("11000"), list("11000"), list("00100"), list("00011")]
    assert s.numIslands(g1) == 1
    assert s.numIslands(g2) == 3
    assert s.numIslands([["0"]]) == 0
    assert s.numIslands([["1"]]) == 1
    assert s.numIslands([list("10101")]) == 3                  # single row
    assert s.numIslands([list("1"), list("0"), list("1")]) == 2  # single column
    assert s.numIslands([list("111"), list("010"), list("111")]) == 1
    assert s.numIslands([list("101"), list("010"), list("101")]) == 5  # diagonals don't connect
    assert g1[0] == list("11110")                              # input untouched

    assert num_islands_recursive(g2) == 3                      # fine on small grids
    big = [["1"] * 300 for _ in range(300)]                    # worst case: one 90k-cell island
    assert s.numIslands(big) == 1
    try:
        num_islands_recursive(big)
        raise AssertionError("expected RecursionError")
    except RecursionError:
        print("recursive DFS on 300x300 all-land grid -> RecursionError (as expected)")
    print("all tests passed")
```

### Dry run: example 2 (3 islands)

| Scan hits | New island? | Flood fill marks | `islands` |
|---|---|---|---|
| (0,0) | ✓ | (0,0) (1,0) (0,1) (1,1) | 1 |
| (0,1), (1,0), (1,1) | ✗ already seen | — | 1 |
| (2,2) | ✓ | (2,2), with no land neighbours | 2 |
| (3,3) | ✓ | (3,3) (3,4) | 3 |
| (3,4) | ✗ already seen | — | 3 |

### Common mistakes

- Marking visited on **dequeue**, which floods the queue with duplicates.
- Recursive DFS without considering depth. It passes LeetCode's tests (whose limits allow it) and then crashes on a production-sized map.
- Counting diagonal neighbours when the problem says horizontal/vertical, or the reverse for 8-connectivity variants.
- `seen = [[False] * cols] * rows`: every row is **the same list object** (Day 1, names are references), so marking one cell marks the whole column.

### Variants to try (optional)

- **LC 695 Max Area of Island:** return the size of each flood fill and keep the max.
- **LC 130 Surrounded Regions:** flood from the *border* first, then everything unmarked is enclosed.
- **LC 417 Pacific Atlantic Water Flow:** two multi-source floods from the oceans inward, then intersect.
- **LC 1091 Shortest Path in Binary Matrix:** BFS with distance layers, 8 directions.

</details>

---

[← Day 26 — Profiling](day-26-profiling.md) · [Day 28 — Consolidation →](day-28-consolidation.md)
