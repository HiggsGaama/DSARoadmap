# Day 23 — Heaps & Top-K: Keeping Only What You Need

[← Week 4](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 10 · attempt 20 · walkthrough 20 · variants 5)

---

## 1. Algorithmic Pattern

A **binary heap** is a complete binary tree stored in a flat array, satisfying the **heap property**: every parent is ≤ its children (min-heap). The root is always the minimum. The rest is only *partially* ordered, and that partial order is what makes it cheaper than sorting.

### How `heapq` works

Python's `heapq` operates **in place on a plain `list`**. There's no heap class.

```text
index:      0    1    2    3    4    5    6
list:     [ 1,   3,   2,   7,   4,   9,   5 ]
parent(i) = (i - 1) // 2      children(i) = 2i + 1, 2i + 2

                 1
             3       2
           7   4   9   5
```

| Operation | Cost | How |
|---|---|---|
| `heap[0]` | $O(1)$ | Peek at the min |
| `heappush(h, x)` | $O(\log n)$ | Append at the end, bubble up while smaller than the parent |
| `heappop(h)` | $O(\log n)$ | Move the last element to the root, then sift down. CPython sifts the hole all the way to a leaf and bubbles back up (Floyd's trick), which needs fewer comparisons on average |
| `heapify(list)` | $O(n)$ | Sift down from the last parent to the root. **Linear, not $n \log n$**, because most nodes sit near the bottom and barely move |
| `heappushpop(h, x)` | $O(\log n)$ | Push, then pop, fused. Returns `x` immediately if `x <= h[0]` |
| `heapreplace(h, x)` | $O(\log n)$ | Pop, then push (always pops the old min) |
| `nlargest(k, it)` / `nsmallest` | $O(n \log k)$ | Uses a size-k heap internally. For `k == 1` use `max`/`min`; for `k` close to `n`, `sorted()` is faster |

**Max-heap:** `heapq` is min-only (3.14 adds public `heapify_max`, `heappush_max` and friends). The portable trick is to **push negated keys**, `(-score, item)`.

### The Top-K pattern: a min-heap of size k for the k largest

Keep a min-heap holding the **k largest items seen so far**. Its root is the *smallest of the top k*, the "bouncer at the door":

- If the heap has fewer than k items, push.
- Otherwise, if the new item beats the root, replace the root (`heapreplace`).
- At the end, the root is the **k-th largest**, and the heap holds the top k (unsorted).

This costs $O(n \log k)$ time and **$O(k)$ memory**, so it works on a stream you can't hold in memory (top-k latencies over a day of logs, top-k retrieval scores in a vector search). Memorize the counterintuitive pairing: **k largest → min-heap; k smallest → max-heap.**

### Preview: two heaps

A max-heap of the lower half plus a min-heap of the upper half, rebalanced so their sizes differ by at most 1, gives a **running median** in $O(\log n)$ per insert (LC 295).

## 2. Identification Rule

Reach for a heap when **any** of these appear:

1. ✅ **"k-th largest/smallest", "top k", "k most frequent", "k closest"**, especially when k ≪ n.
2. ✅ **Streaming / online** input: "data arrives continuously", "design a class with `add()`".
3. ✅ You repeatedly need the **current min/max of a changing set**: schedulers, merging k sorted lists, event simulation, Dijkstra (Week 5).
4. ✅ **Median of a stream** → two heaps.

**Anti-signals:**

- ❌ You need **all** elements sorted → just sort.
- ❌ A one-off k-th element on a static array where average $O(n)$ matters → **quickselect** (below).
- ❌ Values are small bounded integers (frequencies ≤ n) → **bucket sort** gives $O(n)$ (LC 347 trick).

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Kth Largest Element in an Array` | **NeetCode** | Heap vs quickselect, with the quickselect partition drawn out |
| `take U forward heap data structure` | **take U forward** (Striver) | Array representation, sift up/down, heapify from scratch |
| `NeetCode Find Median from Data Stream` | **NeetCode** | Two-heaps preview (LC 295) |

---

## 4. Daily Problem — LeetCode 215: Kth Largest Element in an Array (Medium)

**Statement:** Given an integer array `nums` and an integer `k`, return the **k-th largest element** in sorted order. This is not the k-th *distinct* element: duplicates count. Can you solve it without sorting?

```text
Input:  nums = [3, 2, 1, 5, 6, 4], k = 2
Output: 5

Input:  nums = [3, 2, 3, 1, 2, 4, 5, 5, 6], k = 4
Output: 4            # sorted desc: 6, 5, 5, 4, ...
```

**Constraints:** `1 <= k <= len(nums) <= 10^5`, `-10^4 <= nums[i] <= 10^4`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) If you could keep only k numbers, which k? (2) Which of those k would you need instant access to, to decide whether a newcomer gets in?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **Sort and index:** `sorted(nums)[-k]`. That's $O(n \log n)$ and correct, but it does far more work than needed: it orders all n elements when you only care about the boundary of the top k.
2. **Keep the k largest seen so far** in a min-heap. The root is the smallest of them, so a newcomer only gets in if it beats the root, in which case it replaces the root.
3. After scanning everything, the heap holds exactly the k largest elements, and its **root is the k-th largest**.
4. **Duplicates** need no special handling: "beats the root" means strictly greater. Equal values are interchangeable for the answer.
5. **Quickselect alternative:** partition around a pivot as in quicksort, but recurse only into the side containing index `n − k`. That's average $O(n)$. Use a random pivot and a 3-way partition, because LeetCode includes adversarial inputs with many duplicates that make a naive 2-way partition quadratic.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

| Approach | Time | Extra space | When to use |
|---|---|---|---|
| Sort | $O(n \log n)$ | $O(n)$ | One-liner, small n |
| **Min-heap of size k** | $O(n \log k)$ | $O(k)$ | Streams; k ≪ n; the default interview answer |
| Quickselect (random, 3-way) | $O(n)$ average, $O(n^2)$ worst | $O(n)$ here (list partitions) | Static array, average-case speed matters |

### Code

```python
from __future__ import annotations

import heapq
import random


class Solution:
    def findKthLargest(self, nums: list[int], k: int) -> int:
        """Min-heap of size k: the root is the smallest of the k largest seen so far."""
        heap = nums[:k]
        heapq.heapify(heap)                 # O(k)
        for x in nums[k:]:
            if x > heap[0]:                 # beats the "bouncer" -> gets in
                heapq.heapreplace(heap, x)  # pop root + push x in ONE sift: O(log k)
        return heap[0]

    def findKthLargestQuickselect(self, nums: list[int], k: int) -> int:
        """Average O(n). 3-way partition keeps duplicate-heavy inputs linear."""
        target = k                           # we want the k-th largest
        candidates = nums
        while True:
            pivot = random.choice(candidates)
            greater = [x for x in candidates if x > pivot]
            equal_count = sum(1 for x in candidates if x == pivot)
            if target <= len(greater):
                candidates = greater                     # answer is among the bigger ones
            elif target <= len(greater) + equal_count:
                return pivot                             # answer is the pivot value itself
            else:
                target -= len(greater) + equal_count     # skip the bigger + equal block
                candidates = [x for x in candidates if x < pivot]


if __name__ == "__main__":
    s = Solution()
    cases = [
        ([3, 2, 1, 5, 6, 4], 2, 5),
        ([3, 2, 3, 1, 2, 4, 5, 5, 6], 4, 4),
        ([1], 1, 1),
        ([2, 1], 2, 1),
        ([-1, -1, -1], 2, -1),
        ([7] * 50_000 + [1] * 50_000, 50_001, 1),   # duplicate-heavy adversarial case
    ]
    for nums, k, expected in cases:
        assert s.findKthLargest(nums, k) == expected, (nums[:10], k)
        assert s.findKthLargestQuickselect(nums, k) == expected, (nums[:10], k)
    rng = random.Random(0)
    for _ in range(200):                              # randomized cross-check against sorting
        arr = [rng.randint(-20, 20) for _ in range(rng.randint(1, 40))]
        k = rng.randint(1, len(arr))
        want = sorted(arr, reverse=True)[k - 1]
        assert s.findKthLargest(arr, k) == want == s.findKthLargestQuickselect(arr, k)
    print("all tests passed")
```

### Dry run (heap): `[3, 2, 3, 1, 2, 4, 5, 5, 6]`, `k = 4`

| Step | `x` | Heap before (as list) | `x > root`? | Heap after |
|---|---|---|---|---|
| init | — | heapify `[3,2,3,1]` | — | `[1,2,3,3]` |
| 1 | 2 | root 1 | ✓ replace | `[2,2,3,3]` |
| 2 | 4 | root 2 | ✓ replace | `[2,3,3,4]` |
| 3 | 5 | root 2 | ✓ replace | `[3,3,5,4]` |
| 4 | 5 | root 3 | ✓ replace | `[3,4,5,5]` |
| 5 | 6 | root 3 | ✓ replace | `[4,5,5,6]` |

The root is **4**, the 4th largest.

### Common mistakes

- Using a **max-heap of all n** and popping k times. That's $O(n + k \log n)$, which is fine, but it uses $O(n)$ memory and misses the point for streams.
- Writing `heappush` + `heappop` separately when `heapreplace` (or `heappushpop`) does it in one sift.
- **Tuples with uncomparable payloads:** `heappush(h, (score, request_dict))` raises `TypeError` when two scores tie, because Python then compares the dicts. Use `(score, seq, payload)`, where `seq` comes from an `itertools.count()` tiebreaker. This bug shows up in production schedulers all the time.
- Naive quickselect (first element as pivot, 2-way partition) times out on sorted or all-equal inputs.

### Variants to try (optional)

- **LC 347 Top K Frequent Elements:** `Counter` + size-k heap on `(freq, value)`, or bucket sort by frequency for $O(n)$.
- **LC 703 Kth Largest Element in a Stream:** exactly today's heap, wrapped in a class.
- **LC 295 Find Median from Data Stream:** two heaps (lower half as a negated max-heap, upper half as a min-heap).
- **LC 23 Merge k Sorted Lists:** heap of `(value, list_index, node)` heads.

</details>

---

[← Day 22 — Cancellation](day-22-cancellation-structured-concurrency.md) · [Day 24 — Backpressure & Micro-batching →](day-24-backpressure-micro-batching.md)
