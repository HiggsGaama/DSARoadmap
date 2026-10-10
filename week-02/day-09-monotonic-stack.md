# Day 9 — Monotonic Stack: Resolving "Next Greater" Queries in One Pass

[← Week 2](README.md) · **Track:** DSA · **Time box:** 50 min (pattern 10 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

A plain stack is LIFO storage. A **monotonic stack** keeps its contents sorted (strictly increasing or decreasing from bottom to top) by **popping everything that would break the order before pushing**. The useful part is the pop: when element `x` pops element `y`, you've just learned a fact about `y`, namely that **`x` is the first element after `y` that is greater (or smaller) than it.**

### Mechanics: next greater element

Store **indices** (you usually need distances or positions), with values strictly decreasing from bottom to top:

```text
for i, x in enumerate(a):
    while stack and a[stack[-1]] < x:     # x is the "next greater" for everything it beats
        j = stack.pop()
        answer[j] = i                     # or i - j, or x, depending on the question
    stack.append(i)                       # i waits until something bigger shows up
# indices left on the stack have no next greater element
```

**Why it's $O(n)$:** each index is pushed exactly once and popped at most once. The nested `while` does at most $n$ pops in total over the whole run.

**Why it's correct:** the stack holds exactly the elements that are still "waiting" for an answer. They're in decreasing order because a waiting element that is smaller than a later waiting element would already have been resolved by it.

### Choosing the variant

| Question | Stack order (bottom → top) | Pop when |
|---|---|---|
| Next **greater** to the right | Decreasing | `a[top] < x` |
| Next **smaller** to the right | Increasing | `a[top] > x` |
| Previous greater / smaller (to the left) | Same as above | After popping, the **new top** is the answer for `x` |
| Circular array | Same, but iterate `2n` indices using `i % n` | — |

Strict (`<`) vs non-strict (`<=`) comparison decides how **equal** values are treated. It matters for problems that count spans or subarrays, where duplicates otherwise get double-counted.

## 2. Identification Rule

Reach for a monotonic stack when:

1. ✅ The statement asks for the **next / previous / nearest greater or smaller** element, or "how many days/steps until" something bigger or smaller appears.
2. ✅ You need, for each element, **how far it extends** as a max or min. Examples are stock span, largest rectangle in a histogram, trapping rain water, and sum of subarray minimums.
3. ✅ The brute force is "for each `i`, scan right until a condition on values holds", which is $O(n^2)$.
4. ✅ You're asked to **remove elements to make a sequence lexicographically smallest/largest** (Remove K Digits, Smallest Subsequence of Distinct Characters).

**Anti-signals:**

- ❌ You need the max or min of a **sliding window of fixed size** → monotonic **deque** (pop from both ends). It's a close relative, covered with heaps in Week 4.
- ❌ You need the k-th largest overall → heap.
- ❌ Matching brackets or evaluating expressions → plain stack, not monotonic.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Daily Temperatures` | **NeetCode** | Clean visualization of the decreasing stack resolving waiting days |
| `take U forward next greater element monotonic stack` | **take U forward** (Striver) | Stack/queue playlist: NGE I/II, previous smaller, histogram |
| `NeetCode Largest Rectangle in Histogram` | **NeetCode** | Stretch: the hardest common application of the pattern |

---

## 4. Daily Problem — LeetCode 739: Daily Temperatures (Medium)

**Statement:** Given an array `temperatures` of daily temperatures, return an array `answer` where `answer[i]` is the number of days you have to wait after day `i` to get a **warmer** temperature. If there's no future day for which this is possible, `answer[i] = 0`.

```text
Input:  temperatures = [73, 74, 75, 71, 69, 72, 76, 73]
Output:                [ 1,  1,  4,  2,  1,  1,  0,  0]

Input:  temperatures = [30, 40, 50, 60]
Output:                [ 1,  1,  1,  0]

Input:  temperatures = [30, 60, 90]
Output:                [ 1,  1,  0]
```

**Constraints:** `1 <= len(temperatures) <= 10^5`, `30 <= temperatures[i] <= 100`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) The brute force scans right from each day. Which days are still "waiting" at any moment? (2) What order are the waiting days' temperatures in?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **Brute force:** for each `i`, scan `j = i+1…` until `t[j] > t[i]`. That's $O(n^2)$, and a strictly decreasing input of length $10^5$ means about $5 \times 10^9$ comparisons.
2. **Flip the viewpoint.** Instead of each day searching for its answer, let each *new* day **answer all the earlier days it's warmer than**.
3. **Keep the unanswered days on a stack.** Their temperatures are non-increasing from bottom to top: if a later waiting day were warmer than an earlier waiting day, it would have answered it already.
4. **For each new day `i`,** pop every waiting day `j` with `t[j] < t[i]` and set `answer[j] = i − j`. Then push `i`, which now waits.
5. Days still on the stack at the end never found a warmer day, so they keep the default `0`.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n)$. Each index is pushed once and popped at most once.
- **Space:** $O(n)$ for the stack in the worst case (strictly decreasing input), plus the output.

### Code

```python
from __future__ import annotations


class Solution:
    def dailyTemperatures(self, temperatures: list[int]) -> list[int]:
        n = len(temperatures)
        answer = [0] * n          # default 0 = "no warmer day ahead"
        waiting: list[int] = []   # indices; temperatures non-increasing bottom -> top

        for i, temp in enumerate(temperatures):
            # Today resolves every waiting day that is strictly colder.
            # Strict `<`: an equal temperature is NOT warmer, so that day keeps waiting.
            while waiting and temperatures[waiting[-1]] < temp:
                j = waiting.pop()
                answer[j] = i - j
            waiting.append(i)

        return answer


if __name__ == "__main__":
    s = Solution()
    assert s.dailyTemperatures([73, 74, 75, 71, 69, 72, 76, 73]) == [1, 1, 4, 2, 1, 1, 0, 0]
    assert s.dailyTemperatures([30, 40, 50, 60]) == [1, 1, 1, 0]
    assert s.dailyTemperatures([30, 60, 90]) == [1, 1, 0]
    assert s.dailyTemperatures([50]) == [0]
    assert s.dailyTemperatures([70, 70, 70, 71]) == [3, 2, 1, 0]       # equal is not warmer
    assert s.dailyTemperatures([90, 80, 70, 60]) == [0, 0, 0, 0]       # strictly decreasing
    big = list(range(100, 30, -1)) * 1000 + [101]                      # worst-case stack depth
    assert s.dailyTemperatures(big)[0] == len(big) - 1
    print("all tests passed")
```

### Dry run: `[73, 74, 75, 71, 69, 72, 76, 73]`

| `i` | temp | Pops (`j` → `answer[j]`) | Stack after (temps) |
|---|---|---|---|
| 0 | 73 | — | [73] |
| 1 | 74 | 0 → 1 | [74] |
| 2 | 75 | 1 → 1 | [75] |
| 3 | 71 | — | [75, 71] |
| 4 | 69 | — | [75, 71, 69] |
| 5 | 72 | 4 → 1, 3 → 2 | [75, 72] |
| 6 | 76 | 5 → 1, 2 → 4 | [76] |
| 7 | 73 | — | [76, 73] |

End: indices 6 and 7 remain → `0`. Result `[1, 1, 4, 2, 1, 1, 0, 0]`.

### Common mistakes

- Storing **values** instead of indices. Then you can't compute the distance `i − j`.
- Using `<=` in the pop condition. Equal temperatures would then count as "warmer".
- Iterating backwards without changing the logic. The right-to-left variant exists (keep a stack of *candidate* warmer days and pop the ones that are `<=` the current day), but don't mix the two.

### Variants to try (optional)

- **LC 496 Next Greater Element I:** same stack, plus a hash map from value to answer.
- **LC 503 Next Greater Element II (circular):** loop `i` over `range(2 * n)` and use `nums[i % n]`; only push during the first pass.
- **LC 901 Online Stock Span:** a "previous greater" stack storing `(price, span)` pairs.
- **LC 84 Largest Rectangle in Histogram:** an increasing stack. When a bar pops, its rectangle's width is bounded by the new top and the current index.

</details>

---

[← Day 8 — Context Managers](day-08-context-managers-exceptions.md) · [Day 10 — Descriptors →](day-10-descriptors-attribute-lookup.md)
