# Day 20 — BST Invariants: Propagating Bounds Down the Tree

[← Week 3](README.md) · **Track:** DSA · **Time box:** 50 min (pattern 10 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

A **binary search tree** has one invariant, and it's a statement about **entire subtrees**, not just children:

> For every node `n`: **all** values in `n.left`'s subtree are `< n.val`, and **all** values in `n.right`'s subtree are `> n.val`.

Two equivalent views of that invariant give you two techniques:

### View 1: every node lives inside an interval (top-down bounds)

The root may hold any value: $(-\infty, +\infty)$. Going **left** from a node with value `v` narrows the interval's **upper** bound to `v`. Going **right** narrows the **lower** bound to `v`. Each node must lie strictly inside the interval it inherited:

```text
                    5  (-inf, +inf)
                  /   \
     (-inf, 5)  4       6  (5, +inf)
                      /   \
             (5, 6)  3     7  (6, +inf)
                     ^ 3 is NOT in (5, 6) -> invalid, even though 3 < 6 (its parent)
```

This is **top-down DFS** (Day 16's first variant): the state flows down as arguments.

### View 2: in-order traversal is sorted

Left subtree, node, right subtree. For a BST, the in-order sequence is **strictly increasing**. Many BST problems reduce to "walk in-order and do something with the sorted stream": k-th smallest, validate, two-sum-in-BST, recover a swapped pair, convert to a sorted list. An iterative in-order traversal with an explicit stack can **stop early**, which recursion makes awkward.

### Why BSTs matter outside interviews

The same invariant powers B-trees and B+trees (database indexes), skip lists (Redis sorted sets) and the sorted containers you'd use for range queries. Python's stdlib has no balanced BST. For sorted-order needs use `bisect` on a list, `heapq`, or the `sortedcontainers` package.

An **unbalanced** BST built from sorted input degenerates into a linked list: $O(n)$ operations and $O(n)$ recursion depth. That's why the solution below is iterative.

## 2. Identification Rule

Use BST reasoning when:

1. ✅ The input is explicitly a **BST**. Always ask yourself what the ordering property lets you skip or prune.
2. ✅ **Validate / recover / check** questions about the ordering → bounds propagation, or in-order + `prev`.
3. ✅ **k-th smallest/largest, rank, successor/predecessor, closest value, range sum** → in-order (or reverse in-order), stopping early.
4. ✅ **Search/insert/LCA in a BST** → walk *one* path using comparisons. That's $O(h)$, not $O(n)$.

**Anti-signals:**

- ❌ A plain binary tree (no ordering stated) → the BST shortcuts are invalid. Fall back to general DFS/BFS.
- ❌ "Check each node against its parent and children" → this is the classic **wrong** approach. The invariant concerns ancestors.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Validate Binary Search Tree` | **NeetCode** | Bounds propagation with the grandchild counter-example drawn out |
| `take U forward check if a tree is BST` | **take U forward** (Striver) | Range approach plus the in-order approach |
| `take U forward BST playlist` | **take U forward** | Search, insert, delete, LCA, k-th smallest: the BST toolkit |

---

## 4. Daily Problem — LeetCode 98: Validate Binary Search Tree (Medium)

**Statement:** Given the `root` of a binary tree, determine whether it is a valid BST: every node's left subtree contains only values **strictly less** than the node's value, its right subtree contains only values **strictly greater**, and both subtrees are themselves BSTs.

```text
Input:  root = [2, 1, 3]
Output: true

Input:  root = [5, 1, 4, None, None, 3, 6]
Output: false        # 4 is the root's right child but 4 < 5

Input:  root = [5, 4, 6, None, None, 3, 7]
Output: false        # 3 sits in 5's right subtree (left child of 6) but 3 < 5
```

**Constraints:** `1 <= number of nodes <= 10^4`, `-2^31 <= Node.val <= 2^31 - 1`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) Why does `[5, 4, 6, None, None, 3, 7]` fail if every parent/child pair looks fine? (2) What *range* of values is legal at each node?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **The naive check** compares each node only with its children. It passes `[5, 4, 6, None, None, 3, 7]`, because `3 < 6` is fine locally, but `3` is in the right subtree of `5`. The constraint comes from an **ancestor**.
2. **Carry the constraint down.** Each node inherits an open interval `(low, high)` from its ancestors. Going left, the current value becomes the new `high`. Going right, it becomes the new `low`.
3. A node is valid if `low < val < high`, and the tree is valid if every node is. One pass, $O(n)$.
4. **Bound sentinels:** values span the full 32-bit range, so sentinels like `-2**31` would collide with a legal value. Use `-math.inf` / `math.inf` (or `None` for "unbounded").
5. **Alternative:** the in-order sequence must be strictly increasing, so track `prev` and fail on `val <= prev`.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n)$. Each node is checked once, and both versions can stop at the first violation.
- **Space:** $O(h)$. The in-order stack holds one root-to-node path. The bounds stack holds that path plus at most one pending sibling per level.

### Code

```python
from __future__ import annotations

import math
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
    def isValidBST(self, root: TreeNode | None) -> bool:
        """Top-down bounds propagation, iterative (safe for skewed trees)."""
        # Each entry: (node, low, high) meaning low < node.val < high must hold.
        stack: list[tuple[TreeNode | None, float, float]] = [(root, -math.inf, math.inf)]
        while stack:
            node, low, high = stack.pop()
            if node is None:
                continue
            if not (low < node.val < high):      # strict: duplicates are invalid
                return False
            # Going left, this node becomes the new UPPER bound; the lower bound is inherited.
            stack.append((node.left, low, node.val))
            # Going right, this node becomes the new LOWER bound; the upper bound is inherited.
            stack.append((node.right, node.val, high))
        return True


def is_valid_bst_inorder(root: TreeNode | None) -> bool:
    """Alternative: an in-order traversal of a BST is strictly increasing."""
    stack: list[TreeNode] = []
    node = root
    prev: int | None = None
    while stack or node:
        while node:                              # go as far left as possible
            stack.append(node)
            node = node.left
        node = stack.pop()
        if prev is not None and node.val <= prev:
            return False
        prev = node.val
        node = node.right
    return True


def is_valid_bst_naive(root: TreeNode | None) -> bool:
    """WRONG: only compares each node with its direct children."""
    if root is None:
        return True
    if root.left and root.left.val >= root.val:
        return False
    if root.right and root.right.val <= root.val:
        return False
    return is_valid_bst_naive(root.left) and is_valid_bst_naive(root.right)


if __name__ == "__main__":
    s = Solution()
    cases: list[tuple[list[int | None], bool]] = [
        ([2, 1, 3], True),
        ([5, 1, 4, None, None, 3, 6], False),    # right child 4 < root 5
        ([5, 4, 6, None, None, 3, 7], False),    # 3 is in 5's RIGHT subtree -- grandchild trap
        ([2, 2, 2], False),                      # duplicates violate strict ordering
        ([-2147483648], True),                   # int min: sentinel bounds must not collide
        ([2147483647], True),                    # int max
        ([3, 1, 5, 0, 2, 4, 6], True),
        ([], True),
    ]
    for values, expected in cases:
        root = build_tree(values)
        assert s.isValidBST(root) is expected, values
        assert is_valid_bst_inorder(root) is expected, values

    trap = build_tree([5, 4, 6, None, None, 3, 7])
    print("naive (children only) says [5,4,6,null,null,3,7] is valid:", is_valid_bst_naive(trap))
    print("bounds propagation says:", s.isValidBST(trap))
    print("all tests passed")
```

**Output:**

```text
naive (children only) says [5,4,6,null,null,3,7] is valid: True
bounds propagation says: False
all tests passed
```

### Dry run: `[5, 4, 6, None, None, 3, 7]` (bounds version)

| Pop | Node | Interval | `low < val < high`? |
|---|---|---|---|
| 1 | 5 | (−∞, +∞) | ✓ → push 4 with (−∞, 5), push 6 with (5, +∞) |
| 2 | 6 | (5, +∞) | ✓ → push 3 with (5, 6), push 7 with (6, +∞) |
| 3 | 7 | (6, +∞) | ✓ |
| 4 | 3 | (5, 6) | ✗ **3 ≤ 5 → return False** |

### Common mistakes

- **Comparing only with children** (`is_valid_bst_naive` above): the most common wrong answer.
- **Using `<=` / `>=`** in the bounds. With this problem's strict definition, duplicates are invalid (`[2, 2, 2]` → false).
- **Integer sentinels** like `-2**31` / `2**31 - 1`: a node holding exactly `-2**31` is valid, but the strict check `low < val` against a sentinel equal to it rejects it. Use `math.inf`.
- **In-order with `prev = -inf` initialized as an int sentinel**, which has the same collision problem. Use `None` and check `prev is not None`.

### Variants to try (optional)

- **LC 700 / 701 Search / Insert into a BST:** walk one path, $O(h)$.
- **LC 235 Lowest Common Ancestor of a BST:** the first node whose value lies between `p` and `q` is the LCA.
- **LC 99 Recover Binary Search Tree:** the in-order stream has one or two descents. Find and swap the out-of-place nodes.

</details>

---

[← Day 19 — Typing in Depth](day-19-typing-in-depth.md) · [Day 21 — Consolidation →](day-21-consolidation.md)
