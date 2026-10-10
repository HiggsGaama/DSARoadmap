# Day 18 — Tree BFS: Level-Order Traversal with a Queue

[← Week 3](README.md) · **Track:** DSA · **Time box:** 50 min (pattern 10 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

Breadth-first search visits nodes in order of **distance from the root**: all of depth 0, then all of depth 1, and so on. A FIFO queue does this automatically. Nodes are dequeued in the order they were discovered, and a node's children are always discovered after every node of its own level.

### The level-by-level template

```python
from collections import deque

queue = deque([root])
while queue:
    level_size = len(queue)            # snapshot: exactly the nodes of THIS level
    for i in range(level_size):
        node = queue.popleft()         # O(1) -- never list.pop(0), which is O(n)
        ... process node (i == 0 is leftmost, i == level_size - 1 is rightmost) ...
        if node.left:  queue.append(node.left)
        if node.right: queue.append(node.right)
    ... end of level: record per-level result ...
```

**Why the snapshot matters:** when the inner loop starts, the queue holds exactly one complete level. Children appended during the loop go *behind* them. Reading `len(queue)` once separates the levels without storing a depth on every node.

### BFS vs DFS on trees

| | BFS (queue) | DFS (stack / recursion) |
|---|---|---|
| Visit order | By depth, level by level | One branch to the bottom first |
| Memory | $O(w)$: max width. A complete tree's last level holds about $n/2$ nodes | $O(h)$: height. $O(\log n)$ balanced, $O(n)$ skewed |
| Natural for | Per-level answers, **minimum depth / shortest path** (stops at the first hit) | Subtree aggregates, path enumeration, BST ordering |

On a **wide, shallow** tree (a fan-out-heavy org chart, a file system), BFS can use far more memory than DFS. On a **deep, skewed** tree, DFS recursion hits the recursion limit (Day 16). Pick based on the question, then check the shape.

**Production echo:** the same structure is how you crawl dependency graphs, service call graphs or a paginated hierarchy *by depth*, and how you enforce "max depth N" limits. You stop enqueuing once `level == N`.

## 2. Identification Rule

Use BFS when:

1. ✅ The answer is **per level**: level order, level averages or maxima, zigzag order, largest value in each row.
2. ✅ The words *left side view / right side view / bottom view / row / depth k*.
3. ✅ **Minimum depth**, **nearest** leaf or **fewest steps**: BFS reaches the shallowest answer first and can return immediately. DFS must explore everything.
4. ✅ **Connecting same-level nodes** (Populating Next Right Pointers, LC 116/117).

**Anti-signals:**

- ❌ The answer combines information from entire subtrees (height, diameter, balanced) → post-order DFS (Day 16).
- ❌ Sorted-order questions on a BST → in-order DFS (Day 20).

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Binary Tree Right Side View` | **NeetCode** | BFS with the "last node of each level" observation |
| `take U forward level order traversal` | **take U forward** (Striver) | The queue template in detail, then the views (left, right, top, bottom) |
| `NeetCode Binary Tree Level Order Traversal` | **NeetCode** | 10-minute warm-up: the template on its own |

---

## 4. Daily Problem — LeetCode 199: Binary Tree Right Side View (Medium)

**Statement:** Given the `root` of a binary tree, imagine yourself standing on its **right side**. Return the values of the nodes you can see, ordered from top to bottom.

```text
Input:  root = [1, 2, 3, None, 5, None, 4]      #      1      <- 1
Output: [1, 3, 4]                               #     / \
                                                #    2   3    <- 3
                                                #     \   \
                                                #      5   4  <- 4

Input:  root = [1, 2, 3, 4]                     #      1      <- 1
Output: [1, 3, 4]                               #     / \
                                                #    2   3    <- 3
                                                #   /
                                                #  4          <- 4 (visible: nothing to its right)
```

**Constraints:** `0 <= number of nodes <= 100`, `-100 <= Node.val <= 100`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) The answer has exactly one value per ________. (2) Which node of that group is visible?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. From the right you see exactly **one node per level**: the rightmost node at that depth.
2. **The trap:** "just follow right children". In `[1, 2, 3, 4]` the visible node at depth 2 is `4`, a **left** descendant, because the right subtree is shorter. The rightmost node of a level can come from anywhere.
3. **BFS:** process level by level. The last node dequeued in each level is the answer for that level.
4. **DFS alternative:** visit the **right child first** and track depth. The *first* node reached at each new depth is the visible one. When `depth == len(view)`, you're seeing that depth for the first time.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n)$. Every node is enqueued and dequeued once.
- **Space:** $O(w)$ for the queue, where $w$ is the maximum level width (up to about $n/2$). The DFS version uses $O(h)$.

### Code

```python
from __future__ import annotations

from collections import deque


class TreeNode:
    def __init__(self, val: int = 0, left: TreeNode | None = None, right: TreeNode | None = None) -> None:
        self.val = val
        self.left = left
        self.right = right


def build_tree(values: list[int | None]) -> TreeNode | None:
    """LeetCode-style level-order list -> tree. `None` marks a missing child."""
    if not values or values[0] is None:
        return None
    root = TreeNode(values[0])
    queue = deque([root])
    i = 1
    while queue and i < len(values):
        node = queue.popleft()
        for side in ("left", "right"):
            if i < len(values) and (v := values[i]) is not None:
                child = TreeNode(v)
                setattr(node, side, child)
                queue.append(child)
            i += 1
    return root


class Solution:
    def rightSideView(self, root: TreeNode | None) -> list[int]:
        if root is None:
            return []
        view: list[int] = []
        queue: deque[TreeNode] = deque([root])

        while queue:
            # Snapshot the size: exactly these nodes form the current level.
            # Children appended during the loop belong to the NEXT level.
            level_size = len(queue)
            for i in range(level_size):
                node = queue.popleft()
                if i == level_size - 1:          # last node popped on this level = rightmost
                    view.append(node.val)
                if node.left:
                    queue.append(node.left)
                if node.right:
                    queue.append(node.right)

        return view


def right_side_view_dfs(root: TreeNode | None) -> list[int]:
    """Alternative: DFS visiting RIGHT before LEFT; the first node seen at each depth is visible."""
    view: list[int] = []
    stack: list[tuple[TreeNode, int]] = [(root, 0)] if root else []
    while stack:
        node, depth = stack.pop()
        if depth == len(view):                   # first visit to this depth
            view.append(node.val)
        if node.left:                            # pushed first -> popped after right
            stack.append((node.left, depth + 1))
        if node.right:
            stack.append((node.right, depth + 1))
    return view


if __name__ == "__main__":
    s = Solution()
    cases: list[tuple[list[int | None], list[int]]] = [
        ([1, 2, 3, None, 5, None, 4], [1, 3, 4]),
        ([1, None, 3], [1, 3]),
        ([], []),
        ([1, 2, 3, 4], [1, 3, 4]),               # left subtree is deeper: 4 is visible from the right
        ([1, 2, 3, None, 5, None, None, 6], [1, 3, 5, 6]),
    ]
    for values, expected in cases:
        root = build_tree(values)
        assert s.rightSideView(root) == expected, (values, s.rightSideView(root))
        assert right_side_view_dfs(root) == expected, values
    print("all tests passed")
```

### Dry run: `[1, 2, 3, None, 5, None, 4]` (BFS)

| Level | Queue at level start | Popped in order | Last popped → `view` |
|---|---|---|---|
| 0 | `[1]` | 1 | **1** |
| 1 | `[2, 3]` | 2 (enqueue 5), 3 (enqueue 4) | **3** |
| 2 | `[5, 4]` | 5, 4 | **4** |

### Common mistakes

- Following only right pointers. It fails `[1, 2, 3, 4]`.
- Using `list.pop(0)` as a queue. Each pop is $O(n)$, so the whole traversal is $O(n^2)$ on large trees. Always use `collections.deque`.
- Re-reading `len(queue)` in the loop condition (`while i < len(queue)`) instead of snapshotting it. That mixes levels.
- In the DFS version, pushing right before left onto a **stack**. A stack pops in reverse, so push left first to visit right first.

### Variants to try (optional)

- **LC 102 Binary Tree Level Order Traversal:** collect the whole level instead of the last node.
- **LC 103 Zigzag Level Order:** alternate the direction per level. Use a `deque` for the level and `appendleft` on odd levels.
- **LC 111 Minimum Depth:** BFS that returns at the **first leaf** dequeued, which is often much faster than DFS on unbalanced trees.

</details>

---

[← Day 17 — Dataclasses, Slots & Pydantic](day-17-dataclasses-slots-pydantic.md) · [Day 19 — Typing in Depth →](day-19-typing-in-depth.md)
