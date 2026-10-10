# Day 13 — Fast / Slow Pointers: Floyd's Cycle Detection

[← Week 2](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 15 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

Two pointers walk the same sequence **at different speeds**, usually 1 step and 2 steps. The difference in speed encodes information you'd otherwise need $O(n)$ memory to track:

| Use | Mechanism |
|---|---|
| **Cycle detection** | If there's a cycle, the fast pointer eventually laps the slow one and they meet. If there isn't, fast reaches `None` |
| **Cycle entrance** | After they meet, restart one pointer at the head and move both 1 step at a time. They meet at the entrance (proof below) |
| **Middle of a list** | When fast reaches the end, slow is at the middle |
| **k-th from the end** | A *gap* variant: advance `lead` by k steps, then move both until `lead` hits the end |

The alternative to all of these is "store visited nodes in a set", which is $O(n)$ space. Fast/slow is $O(1)$ space. It's the standard answer whenever the problem says "constant memory".

### Why they must meet

Once both pointers are inside a cycle of length $L$, the gap between them shrinks by exactly 1 node per step (fast gains 1 on slow). A gap of at most $L - 1$ closes in at most $L - 1$ steps, so fast can't jump over slow.

### Why phase 2 finds the entrance

Let $a$ = distance from head to the cycle entrance, $b$ = distance from the entrance to the meeting point (along the cycle), and $L$ = cycle length.

- Slow has walked $a + b$. Fast has walked $2(a + b)$.
- Fast has also walked $a + b + kL$ for some integer $k \ge 1$, since it's done extra laps.
- So $2(a + b) = a + b + kL$, which gives $a + b = kL$, so $a = kL - b = (k - 1)L + (L - b)$.

$L - b$ is the distance from the meeting point forward to the entrance. So a pointer starting at the **head** and a pointer starting at the **meeting point**, both moving 1 step at a time, reach the entrance **at the same moment**: after $a$ steps, the second one has done $k - 1$ full laps plus the final $L - b$.

## 2. Identification Rule

Reach for fast/slow pointers when:

1. ✅ A **linked list** problem mentions **cycle, loop, middle, palindrome, or "k-th from the end"**.
2. ✅ The problem demands **O(1) extra space** where a visited-set would otherwise be the obvious solution.
3. ✅ A sequence is defined by **`x -> f(x)` iteration** and you must detect repetition: Happy Number (sum of squared digits), pseudo-random generators, **Find the Duplicate Number** (treat `i -> nums[i]` as a linked list).
4. ✅ You need to **split a list in half** (for merge sort on a list, or to reverse the second half for a palindrome check).

**Anti-signals:**

- ❌ Random access is available and the data is sorted → binary search (Day 11).
- ❌ Graph cycles with branching (a node has more than one outgoing edge) → DFS with colours or topological sort (Week 5). Floyd needs exactly one successor per node.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Linked List Cycle II` | **NeetCode** | Visual derivation of the $a = (k-1)L + (L - b)$ argument |
| `take U forward starting point of loop in linked list` | **take U forward** (Striver) | Linked-list playlist: middle, cycle, cycle start, palindrome |
| `NeetCode Find the Duplicate Number` | **NeetCode** | Stretch: the same algorithm on an array-as-function-graph |

---

## 4. Daily Problem — LeetCode 142: Linked List Cycle II (Medium)

**Statement:** Given the `head` of a linked list, return **the node where the cycle begins**. If there's no cycle, return `None`. Internally, `pos` is the index of the node the tail's `next` points to (`-1` means no cycle). `pos` isn't passed as a parameter. **Don't modify the list. Follow-up: use O(1) memory.**

```text
Input:  head = [3, 2, 0, -4], pos = 1
Output: node with value 2            # tail (-4) links back to index 1

Input:  head = [1, 2], pos = 0
Output: node with value 1

Input:  head = [1], pos = -1
Output: None
```

**Constraints:** `0 <= number of nodes <= 10^4`, `-10^5 <= Node.val <= 10^5`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) Solve it with a set first: $O(n)$ space. (2) Detect the cycle with two speeds. (3) Write out the distances walked by each pointer at the meeting point.

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **Hash-set solution:** walk the list and return the first node you've already seen. $O(n)$ time and $O(n)$ space. It's correct, but it misses the follow-up.
2. **Phase 1 (detect):** `slow` moves 1 step and `fast` moves 2. If `fast` or `fast.next` becomes `None`, there's no cycle. Otherwise they meet somewhere inside the cycle.
3. **Phase 2 (locate):** the algebra (§1) shows that the distance from the head to the entrance equals the distance from the meeting point to the entrance, modulo whole laps. Reset one pointer to `head`. Move **both by 1** until they're the same node. That node is the entrance.
4. **Compare nodes by identity (`is`), never by value.** Values can repeat.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n)$. Phase 1 takes at most $a + L$ steps for slow, and phase 2 takes $a$ steps.
- **Space:** $O(1)$. Two pointers, no extra structures.

### Code

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass(eq=False)          # eq=False: nodes compare by identity, and dataclass repr won't recurse
class ListNode:
    val: int
    next: ListNode | None = None

    def __repr__(self) -> str:
        return f"ListNode({self.val})"


class Solution:
    def detectCycle(self, head: ListNode | None) -> ListNode | None:
        # Phase 1: detect. fast gains one node per step on slow inside a cycle.
        slow = fast = head
        while fast is not None and fast.next is not None:
            slow = slow.next                  # type: ignore[union-attr]
            fast = fast.next.next
            if slow is fast:
                break
        else:
            return None                       # fast fell off the end: no cycle

        # Phase 2: locate. Distance head->entrance == distance meet->entrance (mod L).
        finder = head
        while finder is not slow:
            finder = finder.next              # type: ignore[union-attr]
            slow = slow.next                  # type: ignore[union-attr]
        return finder


def build(values: list[int], pos: int) -> tuple[ListNode | None, ListNode | None]:
    """Build a list whose tail links to index `pos` (-1 = no cycle). Returns (head, expected)."""
    if not values:
        return None, None
    nodes = [ListNode(v) for v in values]
    for a, b in zip(nodes, nodes[1:]):
        a.next = b
    expected = nodes[pos] if pos >= 0 else None
    nodes[-1].next = expected
    return nodes[0], expected


if __name__ == "__main__":
    s = Solution()
    cases = [
        ([3, 2, 0, -4], 1),
        ([1, 2], 0),
        ([1], -1),
        ([1], 0),                    # self-loop
        ([], -1),
        ([1, 2, 3, 4, 5, 6], -1),
        ([1, 1, 1, 1, 1], 3),        # duplicate values: identity matters
        (list(range(10_000)), 9_999),
        (list(range(10_000)), 0),
    ]
    for values, pos in cases:
        head, expected = build(values, pos)
        assert s.detectCycle(head) is expected, (values[:5], pos)
    print("all tests passed")
```

The `while … else` construct: the `else` branch runs only if the loop ended **without** `break`, meaning `fast` hit the end, so there's no cycle.

### Dry run: `[3, 2, 0, -4]`, `pos = 1` (nodes `n0..n3`, `n3.next = n1`)

Distances: $a = 1$ (head → `n1`), $L = 3$ (`n1 → n2 → n3 → n1`).

| Step | `slow` | `fast` | Note |
|---|---|---|---|
| start | n0 (3) | n0 (3) | |
| 1 | n1 (2) | n2 (0) | |
| 2 | n2 (0) | n1 (2) | fast wrapped: n2 → n3 → n1 |
| 3 | n3 (−4) | n3 (−4) | **meet** at n3, so $b = 2$ |
| phase 2 | `finder`=n0, `slow`=n3 | | |
| 1 | `finder`=n1 | `slow`=n1 | **same node** → return n1 (value 2) ✓ |

Check: $a = (k - 1)L + (L - b) = 0 \cdot 3 + (3 - 2) = 1$ ✓.

### Common mistakes

- Comparing with `==` on values instead of `is`. With duplicate values you return the wrong node. A `@dataclass` with the default `eq=True` would also compare nodes by field values, and recursing through `next` on a cyclic list blows the stack. That's why `eq=False` is used.
- Loop condition `while fast.next and fast.next.next` without first checking `fast`, which crashes on an empty list.
- Phase 2 moving `fast` 2 steps. Both pointers must move 1 step.
- Modifying nodes (marking them visited by changing `val`). That breaks the "don't modify the list" constraint.

### Variants to try (optional)

- **LC 141 Linked List Cycle:** phase 1 only.
- **LC 876 Middle of the Linked List:** when fast stops, slow is the (second) middle.
- **LC 287 Find the Duplicate Number:** treat index `i` as a node and `nums[i]` as its `next`. The duplicate value is the cycle entrance.
- **LC 202 Happy Number:** `x -> sum of squared digits`. Detect whether the sequence reaches 1 or cycles.

</details>

---

[← Day 12 — GIL, Threads & Processes](day-12-gil-threads-processes.md) · [Day 14 — Consolidation →](day-14-consolidation.md)
