# Day 25 — Intervals: Sort by Start, Then Sweep

[← Week 4](README.md) · **Track:** DSA · **Time box:** 50 min (pattern 10 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

Interval problems look 2-dimensional (every interval against every other is $O(n^2)$), but **sorting by start time reduces them to a single linear scan**. After sorting, an interval can only overlap the intervals immediately before it in the sorted order, so you never need to look back further than the "current merged block".

### Overlap: the one predicate to memorize

Two **closed** intervals `[a1, b1]` and `[a2, b2]` overlap iff

$$\max(a1, a2) \le \min(b1, b2)$$

Equivalently, they *don't* overlap iff one ends before the other starts. For half-open intervals `[start, end)`, like meeting times or time ranges in most APIs, use `<` instead of `≤`. **Decide closed vs half-open before writing any code.** Getting this wrong is the most common interval bug, both in interviews and in production (double-booked rooms, gaps in retention windows).

### Three techniques

| Technique | Idea | Classic problems |
|---|---|---|
| **Sort + merge** | Sort by start. Extend the last merged interval if the next one overlaps it, otherwise start a new block | **Merge Intervals (56)**, Insert Interval (57) |
| **Sort + greedy by end** | Sort by **end**. Keep an interval if it starts after the last kept end. This maximizes the number of non-overlapping intervals | Non-overlapping Intervals (435), Min Arrows (452) |
| **Sweep line** | Turn each interval into events `(start, +1)`, `(end, −1)`, sort the events, and keep a running count. The maximum count is the peak concurrency | Meeting Rooms II (253), max concurrent sessions, capacity planning |

Sweep-line tie-breaking encodes the open/closed decision. For half-open intervals, process `−1` before `+1` at the same timestamp: a meeting ending at 10:00 frees the room for one starting at 10:00. Sorting the tuples `(time, delta)` does exactly that, because `−1 < +1`.

**Why sort by start for merging but by end for greedy selection?** Merging needs blocks to be built left to right, so you never discover an earlier interval after closing a block. Maximum selection needs the earliest-*finishing* interval first, because it leaves the most room for the rest (the classic exchange argument).

## 2. Identification Rule

Use interval techniques when **any** of these appear:

1. ✅ Input is a list of **pairs `[start, end]`**: meetings, bookings, time ranges, IP ranges, genomic coordinates, byte ranges.
2. ✅ Words like **merge, overlap, conflict, free slots, coverage, union of ranges, minimum rooms, maximum concurrent**.
3. ✅ "Insert a new range into a sorted set of disjoint ranges".
4. ✅ Platform flavors: merge maintenance windows, compute the union of alert-suppression periods, find peak concurrent connections from start/end logs.

**Which technique:**

- "Merge / union / total covered length" → **sort by start + merge**.
- "Max non-overlapping / min removals / min points to stab all" → **sort by end + greedy**.
- "How many at once / min resources" → **sweep line** (or a min-heap of end times, Day 23).

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Merge Intervals` | **NeetCode** | Sort + merge with the `max(end)` subtlety visualized |
| `NeetCode Meeting Rooms II` | **NeetCode** | Sweep line vs heap, side by side |
| `take U forward merge overlapping subintervals` | **take U forward** (Striver) | Brute force → optimal, with careful edge cases |

---

## 4. Daily Problem — LeetCode 56: Merge Intervals (Medium)

**Statement:** Given an array `intervals` where `intervals[i] = [start_i, end_i]`, merge all overlapping intervals and return an array of the non-overlapping intervals that cover all the intervals in the input. Intervals that **touch** (`[1,4]` and `[4,5]`) count as overlapping.

```text
Input:  intervals = [[1,3],[2,6],[8,10],[15,18]]
Output: [[1,6],[8,10],[15,18]]

Input:  intervals = [[1,4],[4,5]]
Output: [[1,5]]

Input:  intervals = [[1,4],[2,3]]
Output: [[1,4]]          # fully contained interval
```

**Constraints:** `1 <= len(intervals) <= 10^4`, `0 <= start_i <= end_i <= 10^4`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) If the intervals were sorted by start, which previous interval could the current one possibly overlap? (2) When merging, is the new end always the current interval's end?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **Brute force:** repeatedly scan for any overlapping pair and merge it until nothing changes. That's $O(n^2)$ or worse.
2. **Sort by start.** Now every interval that could overlap the current block starts at or after the block's start, and they come *consecutively*. Once an interval starts after the block's end, no later interval can reach back into the block, because later intervals start even later.
3. **Scan with one "open" block** (the last element of the output):
   - If the current interval's start ≤ the block's end, they overlap (closed intervals, touching counts). Extend the block's end to **`max(block_end, cur_end)`**.
   - Otherwise, close the block and start a new one with the current interval.
4. **The `max` matters:** `[1,10]` followed by `[2,3]`. Naively setting the end to 3 would shrink the block. A contained interval must not cut the block short.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n \log n)$ for sorting. The merge scan is $O(n)$.
- **Space:** $O(n)$ for the output, plus the sort's buffer. That's $O(\log n)$ to $O(n)$ depending on the sort implementation, and Timsort can use up to $n/2$.

### Code

```python
from __future__ import annotations


class Solution:
    def merge(self, intervals: list[list[int]]) -> list[list[int]]:
        # Sort by start; ties in start don't matter for correctness.
        ordered = sorted(intervals, key=lambda iv: iv[0])   # don't mutate caller's list
        merged: list[list[int]] = []

        for start, end in ordered:
            if merged and start <= merged[-1][1]:
                # Overlaps (or touches) the open block -> extend it.
                # max() handles intervals fully contained in the block.
                merged[-1][1] = max(merged[-1][1], end)
            else:
                # Gap before this interval -> start a new block (copy: don't alias input).
                merged.append([start, end])

        return merged


def max_concurrency(intervals: list[tuple[int, int]]) -> int:
    """Sweep line over HALF-OPEN [start, end) intervals: peak simultaneous count."""
    events = [(s, +1) for s, _ in intervals] + [(e, -1) for _, e in intervals]
    events.sort()                          # at equal time, -1 sorts before +1: end frees first
    current = peak = 0
    for _, delta in events:
        current += delta
        peak = max(peak, current)
    return peak


if __name__ == "__main__":
    s = Solution()
    assert s.merge([[1, 3], [2, 6], [8, 10], [15, 18]]) == [[1, 6], [8, 10], [15, 18]]
    assert s.merge([[1, 4], [4, 5]]) == [[1, 5]]
    assert s.merge([[1, 4], [2, 3]]) == [[1, 4]]                # contained
    assert s.merge([[1, 10], [2, 3], [4, 5], [11, 12]]) == [[1, 10], [11, 12]]
    assert s.merge([[5, 6], [1, 2]]) == [[1, 2], [5, 6]]        # unsorted input
    assert s.merge([[0, 0]]) == [[0, 0]]
    original = [[2, 3], [1, 2]]
    s.merge(original)
    assert original == [[2, 3], [1, 2]]                        # input not mutated

    assert max_concurrency([(0, 30), (5, 10), (15, 20)]) == 2   # Meeting Rooms II
    assert max_concurrency([(9, 10), (10, 11)]) == 1            # back-to-back: one room
    assert max_concurrency([(1, 5), (2, 6), (3, 7)]) == 3
    print("all tests passed")
```

### Dry run: `[[1,3],[2,6],[8,10],[15,18]]` (already sorted)

| Interval | Open block | `start <= block_end`? | Action | `merged` |
|---|---|---|---|---|
| [1,3] | — | — | new block | `[[1,3]]` |
| [2,6] | [1,3] | 2 ≤ 3 ✓ | end = max(3,6) = 6 | `[[1,6]]` |
| [8,10] | [1,6] | 8 ≤ 6 ✗ | new block | `[[1,6],[8,10]]` |
| [15,18] | [8,10] | 15 ≤ 10 ✗ | new block | `[[1,6],[8,10],[15,18]]` |

### Common mistakes

- `merged[-1][1] = end` instead of `max(...)`, which shrinks blocks around contained intervals.
- Using `<` when touching intervals should merge, or `<=` for half-open ranges that shouldn't.
- Appending the input lists themselves (`merged.append(iv)`) and then mutating them. The caller's data gets silently modified. That's harmless on LeetCode and a real bug in a service that reuses config objects.
- Forgetting to sort, or sorting by end for a merge problem.

### Variants to try (optional)

- **LC 57 Insert Interval:** input already sorted and disjoint. Append the left part, merge the overlapping middle, append the right part, all in $O(n)$.
- **LC 435 Non-overlapping Intervals:** sort by end, greedy. The answer is n − kept.
- **LC 253 Meeting Rooms II:** `max_concurrency` above, or a min-heap of end times.
- **LC 986 Interval List Intersections:** two pointers over two sorted lists. Advance whichever interval ends first.

</details>

---

[← Day 24 — Backpressure & Micro-batching](day-24-backpressure-micro-batching.md) · [Day 26 — Profiling →](day-26-profiling.md)
