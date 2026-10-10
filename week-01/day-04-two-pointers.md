# Day 4 — Two Pointers: Shrinking the Search Space on Sorted Data

[← Week 1](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 10 · attempt 20 · walkthrough 20 · variants 5)

---

## 1. Algorithmic Pattern

Two indices move through a sequence under a rule that **lets you discard a whole row or column of candidate pairs with each step**. That's what turns the $O(n^2)$ pair search into $O(n)$.

### Variant A: opposite ends (converging)

For a **sorted** array and target `T`, start with `lo = 0`, `hi = n − 1`:

- `a[lo] + a[hi] < T`: `a[lo]` paired with *anything* at or below `hi` is too small, because `a[hi]` is the largest value still available. Discard `lo` (`lo += 1`).
- `a[lo] + a[hi] > T`: by the symmetric argument, discard `hi` (`hi -= 1`).
- Equal: record the pair and move both pointers.

**Why it's correct:** picture the $n \times n$ grid of pairs `(i, j)`. Since the array is sorted, the sums increase along each row and each column. Each comparison eliminates an entire row or column, so after at most $n$ steps every cell has been either visited or proven impossible.

### Variant B: same direction (reader/writer, fast/slow)

`write` marks the end of the "kept" prefix and `read` scans ahead. This is how you do in-place compaction (Remove Duplicates, Move Zeroes, Remove Element) and how Floyd cycle detection works on linked lists (Week 2).

### Reduction: k-Sum → fix one element, then two pointers

3Sum is "for each `i`, solve Two Sum II on the suffix with target `−a[i]`". That gives $O(n^2)$. In general, k-Sum is $O(n^{k-1})$.

## 2. Identification Rule

Use Two Pointers when **any** of these appear:

1. ✅ The input is **sorted**, or **you're allowed to sort it** because the output doesn't need original indices or order.
2. ✅ You're looking for **pairs or triplets meeting a sum/difference/product condition**, and moving one end changes the value **monotonically**.
3. ✅ **Palindromes** or **symmetric comparisons** (compare the outermost characters and move inward).
4. ✅ **"In place" / "O(1) extra space"** compaction or partitioning (Dutch National Flag, Move Zeroes).
5. ✅ **"Unique triplets / no duplicate combinations"**. Sorting makes duplicates adjacent, so you can skip them with a neighbour comparison.

**Anti-signals:**

- ❌ You must return **original indices** from unsorted data (classic Two Sum, LC 1): sorting destroys the indices, so use a hash map (Day 2).
- ❌ **Contiguous subarray** with a constraint: use Sliding Window (Day 6). It's also "two pointers", but the invariant concerns the window, not a pair.
- ❌ There's no monotonic relationship between pointer movement and the quantity, for example with negative numbers in a "subarray sum = k" problem. Use prefix sums + hash map.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode 3Sum` | **NeetCode** | Clean visualization of the sort + fix + converge pattern and duplicate skipping |
| `take U forward 3 sum` | **take U forward** (Striver) | Brute → hash set → optimal two pointers, with the dedup logic made explicit |
| `NeetCode Two Sum II input array is sorted` | **NeetCode** | 8-minute warm-up: the inner loop of today's problem |

---

## 4. Daily Problem — LeetCode 15: 3Sum (Medium)

**Statement:** Given an integer array `nums`, return all **unique** triplets `[nums[i], nums[j], nums[k]]` with `i`, `j`, `k` distinct and `nums[i] + nums[j] + nums[k] == 0`. The result must not contain duplicate triplets, in any order.

```text
Input:  nums = [-1, 0, 1, 2, -1, -4]
Output: [[-1, -1, 2], [-1, 0, 1]]

Input:  nums = [0, 1, 1]
Output: []

Input:  nums = [0, 0, 0, 0]
Output: [[0, 0, 0]]
```

**Constraints:** `3 <= len(nums) <= 3000`, `-10^5 <= nums[i] <= 10^5`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) What does sorting buy you for duplicate handling? (2) Fix `nums[i]`. Now what problem remains on the suffix?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **Brute force** is three nested loops plus a set of sorted tuples for dedup: $O(n^3)$. With `n = 3000` that's about $4.5 \times 10^9$ triplets, so it's dead on arrival.
2. **Sort the array.** That gives you (a) monotonic sums for two pointers and (b) duplicates next to each other, so you can skip them with `a[k] == a[k-1]` and never need a set.
3. **Fix the smallest element `a[i]`.** You now need two numbers in `a[i+1:]` summing to `−a[i]`. That's Two Sum II, solved with converging pointers in $O(n)$.
4. **Deduplicate at both levels:**
   - **Outer:** if `a[i] == a[i-1]`, skip it. Every triplet starting with this value was already found.
   - **Inner:** after recording a hit, move `lo` past every copy of the value just used. Moving `hi` past its duplicates isn't needed: once `lo` changes value, the sum check moves `hi` naturally.
5. **Early exit:** if `a[i] > 0`, everything to its right is also positive, so no triplet can sum to 0. Stop.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n^2)$. Sorting is $O(n \log n)$. The outer loop runs $n$ times, and each converging scan is $O(n)$.
- **Space:** $O(1)$ auxiliary, excluding the output and the sort. Python's `sorted()` allocates an $O(n)$ copy and Timsort uses up to $O(n)$ scratch space. Say this explicitly in an interview, because "O(1)" here is commonly overstated.

### Code

```python
from __future__ import annotations


class Solution:
    def threeSum(self, nums: list[int]) -> list[list[int]]:
        a = sorted(nums)  # don't mutate the caller's list (nums.sort() would)
        n = len(a)
        result: list[list[int]] = []

        for i in range(n - 2):
            # Smallest element positive -> all three positive -> sum can't be 0.
            if a[i] > 0:
                break
            # Outer dedup: same first value would reproduce the same triplets.
            if i > 0 and a[i] == a[i - 1]:
                continue

            target = -a[i]
            lo, hi = i + 1, n - 1
            while lo < hi:
                pair_sum = a[lo] + a[hi]
                if pair_sum < target:
                    lo += 1           # need bigger -> discard the smallest candidate
                elif pair_sum > target:
                    hi -= 1           # need smaller -> discard the largest candidate
                else:
                    result.append([a[i], a[lo], a[hi]])
                    lo += 1
                    hi -= 1
                    # Inner dedup: skip repeated values of the second element.
                    while lo < hi and a[lo] == a[lo - 1]:
                        lo += 1

        return result


if __name__ == "__main__":
    def norm(triplets: list[list[int]]) -> list[tuple[int, ...]]:
        return sorted(tuple(t) for t in triplets)

    s = Solution()
    assert norm(s.threeSum([-1, 0, 1, 2, -1, -4])) == [(-1, -1, 2), (-1, 0, 1)]
    assert s.threeSum([0, 1, 1]) == []
    assert s.threeSum([0, 0, 0, 0]) == [[0, 0, 0]]
    assert norm(s.threeSum([-2, 0, 0, 2, 2])) == [(-2, 0, 2)]
    assert norm(s.threeSum([-4, -2, -2, -2, 0, 1, 2, 2, 2, 3, 3, 4, 4, 6, 6])) == [
        (-4, -2, 6), (-4, 0, 4), (-4, 1, 3), (-4, 2, 2),
        (-2, -2, 4), (-2, 0, 2),
    ]
    print("all tests passed")
```

### Dry run: `[-1, 0, 1, 2, -1, -4]` → sorted `[-4, -1, -1, 0, 1, 2]`

| `i` (`a[i]`) | target | (`lo`,`hi`) → sum | Action |
|---|---|---|---|
| 0 (−4) | 4 | (1,5) −1+2=1 <4 → lo++ … (4,5) 1+2=3 <4 → lo++ | lo meets hi: no triplet |
| 1 (−1) | 1 | (2,5) −1+2=1 ✓ | record `[-1,-1,2]`; lo=3, hi=4 |
| | | (3,4) 0+1=1 ✓ | record `[-1,0,1]`; lo=4, hi=3 → stop |
| 2 (−1) | — | — | equals `a[1]` → **skip** (outer dedup) |
| 3 (0) | 0 | (4,5) 1+2=3 >0 → hi-- | lo meets hi: no triplet |

### Common mistakes

- Deduplicating with `set(tuple(...))` on top of two pointers. It works, but it shows you didn't use what sorting gives you. It also costs extra memory and hashing.
- Writing the outer skip as `a[i] == a[i+1]`. That skips the *first* occurrence and misses triplets like `[-1, -1, 2]` that need two copies.
- Forgetting `lo < hi` in the inner dedup `while`, which reads past the end.

### Variants to try (optional)

- **LC 16 3Sum Closest:** same skeleton, track `min(abs(sum − target))`.
- **LC 18 4Sum:** add another outer loop with its own dedup: $O(n^3)$.
- **LC 167 Two Sum II:** the inner loop on its own.

</details>

---

[← Day 3 — Generators](day-03-generators.md) · [Day 5 — asyncio Event Loop →](day-05-asyncio-event-loop.md)
