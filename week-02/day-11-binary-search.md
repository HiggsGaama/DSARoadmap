# Day 11 — Binary Search: Index Search and Search-on-the-Answer

[← Week 2](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 15 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

Binary search isn't really about sorted arrays. It's about a **monotone predicate**: a yes/no function over an ordered range that is `False … False True … True` (it flips exactly once). Binary search finds the **boundary**, the first `True`, in $O(\log R)$ evaluations, where $R$ is the size of the range.

A sorted array is just one source of such a predicate (`a[i] >= target`). Another, more powerful source is the **answer space**: "can the job be done with capacity `k`?" is monotone in `k`, so you can binary-search `k` directly without ever building an array.

### One template to memorize: first `True` in `[lo, hi]`

```python
def first_true(lo: int, hi: int, pred) -> int:
    """Smallest x in [lo, hi] with pred(x) True. Assumes pred(hi) is True and pred is monotone."""
    while lo < hi:
        mid = (lo + hi) // 2      # lower mid: guarantees progress because mid < hi
        if pred(mid):
            hi = mid              # mid might be the answer: keep it
        else:
            lo = mid + 1          # mid is definitely not: discard it
    return lo                     # lo == hi == boundary
```

**Invariant:** the answer is always in `[lo, hi]`. Each step either keeps `mid` (`hi = mid`) or discards it (`lo = mid + 1`). Since `mid < hi`, the range strictly shrinks, so no infinite loop is possible.

Almost every binary-search problem reduces to choosing **(1) the range `[lo, hi]`** and **(2) the predicate**:

| Problem | Range | Predicate |
|---|---|---|
| Lower bound (first index ≥ target) | `[0, n]` | `a[i] >= target` |
| Upper bound (first index > target) | `[0, n]` | `a[i] > target` |
| Min speed / capacity / days | `[min_feasible, max_needed]` | `feasible(k)` |
| Min of rotated sorted array | `[0, n−1]` | `a[i] <= a[n−1]` |
| Integer square root | `[0, x]` | `m * m > x`, then the answer is `first_true − 1` |

> **Python note:** `bisect.bisect_left(a, x)` is lower bound and `bisect.bisect_right` is upper bound. Both take a `key=` argument since 3.10. Use them in production code. Write the template by hand in interviews and for answer-space search.

**Overflow:** Python ints don't overflow. In Java/C++/Go, `(lo + hi) / 2` can, so use `lo + (hi − lo) / 2`. It's worth mentioning in an interview.

## 2. Identification Rule

Reach for binary search when:

1. ✅ The input is **sorted** (or rotated sorted) and you need a position, a boundary, or a count of elements in a range.
2. ✅ The question is **"minimum X such that …"** or **"maximum X such that …"**, and making X larger makes the condition easier (or harder) monotonically. That's a search on the answer.
3. ✅ The constraints have **huge value ranges** (`up to 10^9`) but a feasibility check that costs $O(n)$. That suggests $O(n \log V)$.
4. ✅ The words **"at most H hours / D days / K splits"** together with "minimize the maximum" or "maximize the minimum" (split array, ship packages, aggressive cows).

**Anti-signals:**

- ❌ The predicate isn't monotone. If feasible at `k` doesn't imply feasible at `k+1`, binary search silently returns garbage.
- ❌ Unsorted input where you need one lookup: a hash set is $O(1)$.
- ❌ The answer depends on the arrangement, not a threshold: think DP or greedy.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Koko Eating Bananas` | **NeetCode** | Answer-space binary search explained visually |
| `take U forward binary search on answers playlist` | **take U forward** (Striver) | The most complete series on this pattern: Koko, ship packages, aggressive cows, split array |
| `Errichto binary search` | **Errichto** | Competitive-programming view of the "first true" invariant |

---

## 4. Daily Problem — LeetCode 875: Koko Eating Bananas (Medium)

**Statement:** There are `n` piles of bananas; pile `i` has `piles[i]` bananas. The guards return in `h` hours. Each hour Koko picks one pile and eats `k` bananas from it. If the pile has fewer than `k`, she eats all of it and doesn't eat any more that hour. Return the **minimum integer `k`** that lets her eat all the bananas within `h` hours.

```text
Input:  piles = [3, 6, 7, 11], h = 8
Output: 4

Input:  piles = [30, 11, 23, 4, 20], h = 5
Output: 30

Input:  piles = [30, 11, 23, 4, 20], h = 6
Output: 23
```

**Constraints:** `1 <= len(piles) <= 10^4`, `len(piles) <= h <= 10^9`, `1 <= piles[i] <= 10^9`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) For a fixed `k`, how many hours does she need? (2) If `k` works, does `k + 1` work? (3) What are the smallest and largest `k` worth trying?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **For a fixed speed `k`,** pile `p` takes $\lceil p / k \rceil$ hours, so the total is the sum over piles. That's $O(n)$ to evaluate.
2. **Monotone:** a higher speed never needs more hours. So `feasible(k) = hours(k) <= h` is `False … False True … True` over `k`.
3. **Range:** `k = 1` is the smallest meaningful speed. `k = max(piles)` always works, because each pile then takes exactly 1 hour and `h >= n` is guaranteed. Going faster than that never helps.
4. **Linear scan** over `k` would cost $O(n \cdot \max(piles))$, which is about $10^{13}$. Binary-search for the first feasible `k` instead: $O(n \log \max(piles))$, about $10^4 \times 30$.
5. **Ceiling division without floats:** `(p + k - 1) // k` or `-(-p // k)`. Never use `math.ceil(p / k)` with values up to $10^9$ or larger, because float rounding can be off by one on big inputs.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n \log M)$, where $M = \max(piles)$. There are about $\log M \le 30$ iterations (base 2), each an $O(n)$ feasibility check.
- **Space:** $O(1)$.

### Code

```python
from __future__ import annotations


class Solution:
    def minEatingSpeed(self, piles: list[int], h: int) -> int:
        def hours_needed(k: int) -> int:
            # Integer ceiling division: ceil(p / k) == (p + k - 1) // k
            return sum((p + k - 1) // k for p in piles)

        # Answer space: k in [1, max(piles)]. pred(max(piles)) is guaranteed True (h >= n).
        lo, hi = 1, max(piles)
        while lo < hi:
            mid = (lo + hi) // 2
            if hours_needed(mid) <= h:
                hi = mid          # mid is fast enough; maybe something slower is too
            else:
                lo = mid + 1      # too slow; the answer is strictly faster
        return lo


if __name__ == "__main__":
    s = Solution()
    assert s.minEatingSpeed([3, 6, 7, 11], 8) == 4
    assert s.minEatingSpeed([30, 11, 23, 4, 20], 5) == 30
    assert s.minEatingSpeed([30, 11, 23, 4, 20], 6) == 23
    assert s.minEatingSpeed([1], 1) == 1
    assert s.minEatingSpeed([1_000_000_000], 2) == 500_000_000
    assert s.minEatingSpeed([312884470], 312884469) == 2
    assert s.minEatingSpeed([5, 5, 5], 1_000_000_000) == 1         # tons of time -> slowest speed
    print("all tests passed")
```

### Dry run: `piles = [3, 6, 7, 11], h = 8`

| `lo` | `hi` | `mid` | hours(mid) = ⌈3/m⌉+⌈6/m⌉+⌈7/m⌉+⌈11/m⌉ | ≤ 8? | Update |
|---|---|---|---|---|---|
| 1 | 11 | 6 | 1+1+2+2 = 6 | ✓ | hi = 6 |
| 1 | 6 | 3 | 1+2+3+4 = 10 | ✗ | lo = 4 |
| 4 | 6 | 5 | 1+2+2+3 = 8 | ✓ | hi = 5 |
| 4 | 5 | 4 | 1+2+2+3 = 8 | ✓ | hi = 4 |
| 4 | 4 | — | — | — | return **4** |

### Common mistakes

- `while lo <= hi` combined with `hi = mid`. That's an infinite loop when `lo == hi`. Pair the loop condition with the right update rule: `lo < hi` goes with `hi = mid` / `lo = mid + 1`.
- Starting `lo = 0`, which causes a division by zero in `hours_needed`.
- Float ceiling (`math.ceil(p / k)`). It's correct on the samples but brittle at $10^{9}$ scale and slower.
- Setting `hi = sum(piles)`. It's correct but wastes iterations, since `max(piles)` already suffices.

### Variants to try (optional)

- **LC 1011 Capacity To Ship Packages Within D Days:** range `[max(w), sum(w)]`, with a greedy day counter as the predicate.
- **LC 410 Split Array Largest Sum:** the same predicate shape ("minimize the maximum").
- **LC 34 Find First and Last Position:** two lower-bound searches (`target` and `target + 1`).

</details>

---

[← Day 10 — Descriptors](day-10-descriptors-attribute-lookup.md) · [Day 12 — GIL, Threads & Processes →](day-12-gil-threads-processes.md)
