# Day 2 — Hashing: Trading Memory for O(1) Lookups

[← Week 1](README.md) · **Track:** DSA · **Time box:** 50 min (pattern 10 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

A hash set or hash map turns "have I seen X?" and "how many X?" from an $O(n)$ scan into an expected $O(1)$ lookup. Most brute-force $O(n^2)$ solutions that compare every element with every other element can be rewritten as **one pass that asks the hash table a question**.

Four common shapes:

| Shape | Structure | Question asked per element | Classic problems |
|---|---|---|---|
| **Membership** | `set` | "Is `x` (or its neighbour, `x−1`, `x+1`) present?" | Contains Duplicate, **Longest Consecutive Sequence** |
| **Complement** | `dict` value→index | "Have I seen `target − x`?" | Two Sum, Pair with Given Difference |
| **Frequency** | `Counter` / `dict` | "How many of `x` so far?" | Valid Anagram, Top K Frequent, Majority Element |
| **Grouping by canonical key** | `defaultdict(list)` | "Which bucket does `key(x)` go in?" | Group Anagrams, Group Shifted Strings |

**The cost model:** Python's `dict` and `set` use open addressing with a perturbation-based probe sequence and resize at about 2/3 load. Lookups are expected $O(1)$ and worst-case $O(n)$ under adversarial collisions. String hashing is randomized per process (`PYTHONHASHSEED`) for that reason. Memory overhead is real: a `set` of 1M ints takes about 32 MB of table on top of the int objects.

## 2. Identification Rule

Reach for hashing when **any** of these appear:

1. ✅ **"Unsorted" + "O(n) time required"**. Sorting costs $O(n \log n)$, so the problem setter is telling you to use a hash table.
2. ✅ The brute force is **"for each element, search the rest"** for a value derived from the current element (complement, neighbour, anagram key).
3. ✅ Words like **duplicate, frequency, count, group, anagram, seen before, first unique**.
4. ✅ You need **order-independent equality** of collections (compare `Counter`s, or sorted-tuple keys).

**Anti-signals:**

- ❌ The input is **already sorted** and you need pairs → Two Pointers (Day 4) gives $O(1)$ space.
- ❌ You need **contiguous** subarrays/substrings → Sliding Window (Day 6) or prefix sums. A hash table may still be the *window's state*.
- ❌ You need the k-th smallest or ordered iteration → heap or sorting.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Longest Consecutive Sequence` | **NeetCode** | Visualizes the "only start counting at sequence starts" insight |
| `take U forward longest consecutive sequence` | **take U forward** (Striver) | Walks brute → better (sort) → optimal (set) |
| `NeetCode arrays and hashing playlist` | **NeetCode** | Do Two Sum, Group Anagrams and Top K Frequent as warm-ups this week |

---

## 4. Daily Problem — LeetCode 128: Longest Consecutive Sequence (Medium)

**Statement:** Given an unsorted array of integers `nums`, return the length of the longest sequence of consecutive integers (values `x, x+1, x+2, …`). The elements don't need to be adjacent in the array. **You must write an algorithm that runs in $O(n)$ time.**

```text
Input:  nums = [100, 4, 200, 1, 3, 2]
Output: 4                 # the sequence 1, 2, 3, 4

Input:  nums = [0, 3, 7, 2, 5, 8, 4, 6, 0, 1]
Output: 9                 # 0..8

Input:  nums = []
Output: 0
```

**Constraints:** `0 <= len(nums) <= 10^5`, `-10^9 <= nums[i] <= 10^9`.

> ⏱ Attempt it for 20 minutes before opening anything below. Hint ladder: (1) What would the sorting solution cost? (2) If you could ask "does x−1 exist?" in O(1), which elements would you skip?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **Brute force:** for each `x`, keep checking whether `x+1`, `x+2`, … exist by scanning the array. That's $O(n)$ per check and $O(n^3)$ overall.
2. **Sort first:** sort, then walk once counting runs and skipping duplicates. $O(n \log n)$ is correct, but it breaks the stated $O(n)$ requirement.
3. **Key insight:** put every number in a set, so "does `y` exist?" is $O(1)$. Starting a count from *every* number is wasteful, because from `3` you'd re-walk `3, 4`, which you already walked from `1`.
4. **Count only from sequence starts.** `x` is a start exactly when `x − 1` is **not** in the set. From each start, walk upward while `x + length` is in the set.
5. **Why that is linear:** each number belongs to exactly one maximal run, and only that run's start walks across it. Total inner-loop steps across the whole algorithm ≤ n.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n)$. Building the set is $O(n)$. The outer loop is $O(n)$, and the inner `while` loop runs a combined total of at most $n$ times, because it only advances from run starts and runs are disjoint.
- **Space:** $O(n)$ for the set.

### Code

```python
from __future__ import annotations


class Solution:
    def longestConsecutive(self, nums: list[int]) -> int:
        # O(1) membership checks; also collapses duplicates.
        num_set = set(nums)
        best = 0

        # Iterate the SET, not `nums`. With nums = [1]*50_000 + list(range(2, 50_001)),
        # iterating the list would re-walk the long run from `1` 50,000 times -> O(n^2).
        for x in num_set:
            # Only start counting at the beginning of a run.
            if x - 1 in num_set:
                continue

            length = 1
            while x + length in num_set:
                length += 1

            best = max(best, length)

        return best


if __name__ == "__main__":
    s = Solution()
    assert s.longestConsecutive([100, 4, 200, 1, 3, 2]) == 4
    assert s.longestConsecutive([0, 3, 7, 2, 5, 8, 4, 6, 0, 1]) == 9
    assert s.longestConsecutive([]) == 0
    assert s.longestConsecutive([7]) == 1
    assert s.longestConsecutive([1, 2, 0, 1]) == 3              # duplicates
    assert s.longestConsecutive([-3, -2, -1, 5]) == 3           # negatives
    assert s.longestConsecutive([1] * 50_000 + list(range(2, 50_001))) == 50_000  # dup-heavy run
    print("all tests passed")
```

### Dry run: `[100, 4, 200, 1, 3, 2]`

| `x` | `x−1` in set? | Action | Run length | `best` |
|---|---|---|---|---|
| 100 | 99 ✗ | start → 101 ✗ | 1 | 1 |
| 4 | 3 ✓ | skip | — | 1 |
| 200 | 199 ✗ | start → 201 ✗ | 1 | 1 |
| 1 | 0 ✗ | start → 2 ✓ 3 ✓ 4 ✓ 5 ✗ | 4 | **4** |
| 3 | 2 ✓ | skip | — | 4 |
| 2 | 1 ✓ | skip | — | 4 |

(Set iteration order differs in practice; the result doesn't depend on it.)

### Common mistakes

- **Iterating `nums` instead of `num_set`.** It's correct on small inputs, but a test with many duplicates of a run's start times it out. This is the most common TLE on this problem.
- Using `sorted()` and calling it "basically linear". Interviewers notice.
- Off-by-one: the counter starts at `length = 1`, because the start counts.

### Variants to try (optional)

- **Union-Find version** (Week 5): union `x` with `x+1` and track component sizes. It's overkill here, but it's the same problem.
- **Return the sequence itself:** track `best_start` and output `range(best_start, best_start + best)`.
- **Streaming:** numbers arrive one at a time and you must report the current longest run after each. Use a `dict` mapping run boundaries to lengths and merge the left and right neighbours in $O(1)$ per insert.

</details>

---

[← Day 1 — Memory Model](day-01-cpython-memory-model.md) · [Day 3 — Generators →](day-03-generators.md)
