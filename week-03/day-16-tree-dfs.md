# Day 16 — Tree DFS: Return Values, Global Updates, and Explicit Stacks

[← Week 3](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 15 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

Depth-first search on a tree explores one branch to the bottom before backtracking. The real skill isn't the traversal itself, which is three lines. It's deciding **what each recursive call returns** and **what gets updated on the side**.

### The core template: post-order "ask children, then decide"

```python
def dfs(node) -> Info:
    if node is None:
        return BASE                     # the empty tree's answer
    left = dfs(node.left)               # trust the recursion: children are solved
    right = dfs(node.right)
    # (optional) global update using BOTH children -- a path that bends here
    return combine(node, left, right)   # what the PARENT needs -- usually ONE branch
```

Three variants cover most tree problems:

| Variant | Information flows | Example problems |
|---|---|---|
| **Top-down (pre-order)** | Parent → child via **arguments** (depth so far, running sum, allowed range) | Path Sum, Max Depth (top-down), **Validate BST** (Day 20) |
| **Bottom-up (post-order)** | Child → parent via **return values** | Height, Balanced Binary Tree, subtree sums |
| **Bottom-up + global** | Return value for the parent, *plus* a `nonlocal` answer updated at every node | **Diameter**, Binary Tree Maximum Path Sum (LC 124), Longest Univalue Path |

**The key distinction in the third variant:** a path that *bends* at a node (left branch + node + right branch) is a candidate **answer**, but it can't be **returned**, because the parent can extend only one straight branch. So you update the global answer with the bend and return the straight branch.

### Recursion depth is a production concern

CPython's default recursion limit is **1000** (`sys.getrecursionlimit()`). A balanced tree of a billion nodes is only about 30 levels deep, but a **skewed** tree (sorted inserts into a naive BST, a parent-pointer chain from user data, a deeply nested JSON document) can be as deep as $n$. Raising the limit with `sys.setrecursionlimit` only moves the cliff. Each Python frame still costs memory, and when the recursion goes through C code (`repr`, `json`, `copy.deepcopy`, C extensions) a high limit can overflow the C stack and crash the process instead of raising `RecursionError`. For trees of unbounded depth, **convert to an explicit stack**. The pattern is mechanical: push `(node, state)` tuples, where `state` records whether the children have been processed yet.

## 2. Identification Rule

Reach for tree DFS when:

1. ✅ The answer for a node can be computed **from the answers of its subtrees**: height, size, sum, balanced, max path, "does the subtree contain X". This is post-order.
2. ✅ The problem constrains **root-to-node paths**: path sum, valid range, depth so far. Pass state *down* as arguments (pre-order).
3. ✅ You need **every root-to-leaf path** or must **backtrack** (Path Sum II): DFS with a path list you append to and pop from.
4. ✅ The words *diameter*, *longest path between any two nodes*, *maximum path sum* signal **return one branch, update a global with both**.

**Anti-signals:**

- ❌ "Level", "row", "minimum depth", "closest to root", "left/right side view" → **BFS** (Day 18) is more natural, and stops early for minimum-depth problems.
- ❌ Ordering questions on a **BST** (k-th smallest, range sum) → in-order traversal specifically (Day 20/21).

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Diameter of Binary Tree` | **NeetCode** | Explains the "height vs. diameter" separation visually |
| `take U forward diameter of binary tree` | **take U forward** (Striver) | Part of his tree series; shows the brute $O(n^2)$ → optimal $O(n)$ progression |
| `take U forward iterative postorder traversal` | **take U forward** | Converting recursion to explicit stacks |

---

## 4. Daily Problem — LeetCode 543: Diameter of Binary Tree (Easy, but a pattern-defining one)

**Statement:** Given the `root` of a binary tree, return the length of its **diameter**: the number of **edges** on the longest path between any two nodes. The path **may or may not pass through the root**.

```text
Input:  root = [1, 2, 3, 4, 5]          #        1
Output: 3                               #       / \
                                        #      2   3
# path 4 -> 2 -> 1 -> 3 (3 edges)       #     / \
                                        #    4   5

Input:  root = [1, 2]
Output: 1
```

**Constraints:** `1 <= number of nodes <= 10^4`, `-100 <= Node.val <= 100`.

Trees use LeetCode's level-order list notation, where `None` marks a missing child. The solution below includes a `build_tree` helper so you can test locally.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) For one fixed node, what's the longest path that *bends* there? (2) Which quantity do you need from each child to compute that?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. Every path in a tree has exactly one **highest node**, where it "bends". So the diameter is the maximum, over all nodes, of the longest path bending at that node.
2. The longest path bending at `node` = (height of left subtree) + (height of right subtree), where height counts the nodes on the longest downward path. That many nodes below on each side means that many edges through `node`.
3. **Brute force:** compute heights separately for every node. That's $O(n)$ per node and $O(n^2)$ overall on skewed trees.
4. **Optimal:** a single post-order pass. Each call **returns its height** to its parent and, on the side, **updates `best`** with `left + right`. Every node's bend is checked once.
5. **The trap:** returning `left + right` (the bend) to the parent. The parent can only extend one branch, so the return value must be `1 + max(left, right)`.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n)$. Each node is visited once with $O(1)$ work.
- **Space:** $O(h)$ for the recursion or the explicit stack, where $h$ is the tree height: $O(\log n)$ when balanced and $O(n)$ when skewed.

### Code

```python
from __future__ import annotations

import sys
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
    def diameterOfBinaryTree(self, root: TreeNode | None) -> int:
        best = 0  # longest path seen anywhere, measured in EDGES

        def height(node: TreeNode | None) -> int:
            """Number of NODES on the longest downward path starting at `node`."""
            nonlocal best
            if node is None:
                return 0
            left = height(node.left)
            right = height(node.right)
            # Global update: the longest path that BENDS at this node uses
            # `left` edges down the left side and `right` edges down the right side.
            best = max(best, left + right)
            # Return value: what the PARENT needs -- one straight branch, not the bend.
            return 1 + max(left, right)

        height(root)
        return best


def diameter_iterative(root: TreeNode | None) -> int:
    """Same algorithm with an explicit stack: post-order, no recursion limit."""
    if root is None:
        return 0
    heights: dict[int, int] = {}  # id(node) -> height (nodes on longest downward path)
    best = 0
    stack: list[tuple[TreeNode, bool]] = [(root, False)]
    while stack:
        node, children_done = stack.pop()
        if not children_done:
            stack.append((node, True))  # revisit after both children are finished
            if node.right:
                stack.append((node.right, False))
            if node.left:
                stack.append((node.left, False))
            continue
        left = heights.pop(id(node.left), 0) if node.left else 0
        right = heights.pop(id(node.right), 0) if node.right else 0
        best = max(best, left + right)
        heights[id(node)] = 1 + max(left, right)
    return best


if __name__ == "__main__":
    s = Solution()
    cases: list[tuple[list[int | None], int]] = [
        ([1, 2, 3, 4, 5], 3),                                   # 4-2-1-3 or 5-2-1-3
        ([1, 2], 1),
        ([1], 0),
        ([], 0),
        # Diameter does NOT pass through the root here: it lives in the left subtree.
        ([1, 2, None, 3, 4, 5, None, None, 6, 7, None, None, 8], 6),
    ]
    for values, expected in cases:
        root = build_tree(values)
        assert s.diameterOfBinaryTree(root) == expected, values
        assert diameter_iterative(root) == expected, values

    # A degenerate (linked-list shaped) tree deeper than the recursion limit.
    deep = TreeNode(0)
    node = deep
    for i in range(1, 50_000):
        node.left = TreeNode(i)
        node = node.left
    try:
        s.diameterOfBinaryTree(deep)
    except RecursionError:
        print(f"recursive version: RecursionError (limit={sys.getrecursionlimit()})")
    print("iterative version on 50,000-deep tree:", diameter_iterative(deep))
    print("all tests passed")
```

**Output:**

```text
recursive version: RecursionError (limit=1000)
iterative version on 50,000-deep tree: 49999
all tests passed
```

### Dry run: `[1, 2, 3, 4, 5]`

| Node (post-order) | `left` | `right` | Bend `left + right` | `best` | Returns `1 + max` |
|---|---|---|---|---|---|
| 4 | 0 | 0 | 0 | 0 | 1 |
| 5 | 0 | 0 | 0 | 0 | 1 |
| 2 | 1 | 1 | 2 | 2 | 2 |
| 3 | 0 | 0 | 0 | 2 | 1 |
| 1 | 2 | 1 | **3** | **3** | 3 |

### Common mistakes

- Returning `max(left, right) + 1` but also **returning `best` from the root call** as if it were the diameter. Root height ≠ diameter whenever the longest path lives inside a subtree (test case 5).
- Counting nodes instead of edges, which gives an off-by-one result.
- Using an instance attribute `self.best` without resetting it between calls on the same `Solution` object. `nonlocal` inside the method avoids that state leak.
- In the iterative version, keying a dict by node **object** works (identity hash), but keying by `node.val` breaks on duplicate values.

### Variants to try (optional)

- **LC 110 Balanced Binary Tree:** return height, or `-1` as a sentinel meaning "already unbalanced" to short-circuit.
- **LC 124 Binary Tree Maximum Path Sum (Hard):** the same skeleton with values instead of counts, and `max(0, child)` to drop negative branches.
- **LC 687 Longest Univalue Path:** the same skeleton, extending a branch only when the child's value matches.

</details>

---

[← Day 15 — Metaclasses](day-15-metaclasses-class-creation.md) · [Day 17 — Dataclasses, Slots & Pydantic →](day-17-dataclasses-slots-pydantic.md)
