# Day 6 — Sliding Window: Maintaining an Invariant Over a Contiguous Range

[← Week 1](README.md) · **Track:** DSA · **Time box:** 50 min (pattern 10 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

A window `[left, right]` moves across a sequence. `right` expands it one element at a time, and `left` shrinks it **only as far as needed** to restore validity. Both pointers only move forward, so each element enters and leaves the window at most once. That gives $O(n)$ instead of the $O(n^2)$ cost of checking every subarray.

### Two variants

| Variant | Window size | Template | Examples |
|---|---|---|---|
| **Fixed** | Always `k` | Add `a[right]`, remove `a[right − k]`, record | Max sum of size-k subarray, moving averages, anagram-in-string (LC 438) |
| **Variable** | Grows and shrinks | Expand right; `while invalid: shrink left`; record | **Longest substring w/o repeats**, Min Window Substring (LC 76), Longest Repeating Char Replacement (LC 424) |

### The variable-window template

```python
left = 0
state = ...                       # counts / set / running sum describing the window
for right, x in enumerate(seq):
    add x to state
    while window is invalid:      # for "shortest valid", flip: while window IS valid → record, shrink
        remove seq[left] from state
        left += 1
    record answer using right - left + 1
```

### Why it's correct: monotonicity

The technique depends on one property. **If a window is invalid, every window that contains it is also invalid.** (Equivalently, if a window is valid, every sub-window of it is valid.) "No repeated characters" has this property: adding characters can't remove a duplicate. So once `[left, right]` is invalid, no window starting at `left` and ending at or after `right` can be valid, and advancing `left` permanently loses nothing.

**When monotonicity fails, the window fails.** "Subarray sum equals k" with **negative numbers**: adding an element can *decrease* the sum, so a too-large window might become valid again. Use **prefix sums + hash map** in that case.

## 2. Identification Rule

Use a sliding window when **all three** of these hold:

1. ✅ The answer is a **contiguous** subarray or substring. Look for the words *substring*, *subarray*, *consecutive*, *window*, *within k positions*.
2. ✅ You're asked for the **longest / shortest / max / min / count** of such ranges.
3. ✅ The validity constraint is **monotone**: extending the window can only keep it valid or make it invalid, never fix it. Examples: "at most K distinct", "no repeats", "sum ≤ S with non-negative values", "at most K replacements".

**Quick tests:**

- "Subsequence" (not necessarily contiguous) → **not** a sliding window. Think DP or two pointers on two strings.
- "Exactly K" → compute `atMost(K) − atMost(K − 1)`, where each is a sliding window. This trick is worth memorizing.
- Negative numbers + sum target → prefix sums.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Longest Substring Without Repeating Characters` | **NeetCode** | Visual walk of the set-based window |
| `take U forward sliding window two pointer playlist` | **take U forward** (Striver) | Systematic pattern series: fixed, variable, "at most K", "exactly K" |
| `NeetCode Minimum Window Substring` | **NeetCode** | Preview of the "shortest valid" flip; do it as a stretch problem this week |

---

## 4. Daily Problem — LeetCode 3: Longest Substring Without Repeating Characters (Medium)

**Statement:** Given a string `s`, find the length of the **longest substring** (contiguous) that contains no repeated characters.

```text
Input:  s = "abcabcbb"
Output: 3          # "abc"

Input:  s = "bbbbb"
Output: 1          # "b"

Input:  s = "pwwkew"
Output: 3          # "wke"  ("pwke" is a subsequence, not a substring)

Input:  s = ""
Output: 0
```

**Constraints:** `0 <= len(s) <= 5 * 10^4`; `s` contains English letters, digits, symbols and spaces.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) What makes a window invalid? (2) When `s[right]` is a duplicate, where *exactly* must `left` move to?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **Brute force:** check every substring for uniqueness. That's $O(n^2)$ substrings × $O(n)$ check = $O(n^3)$. With a set grown incrementally per start index it's $O(n^2)$. Still too slow for $n = 5 \times 10^4$ in the worst case.
2. **Window invariant:** `s[left..right]` has no repeats. Extend `right` by one each step.
3. **When `s[right]` = `c` already appears in the window,** the window is invalid. Any valid window ending at `right` must start **after the previous occurrence of `c`**, so `left` moves to `last_seen[c] + 1`.
4. **Jump directly instead of shrinking one step at a time.** Store the last index of each character, and move `left` straight to `last_seen[c] + 1`.
5. **The trap:** `last_seen[c]` might point to a position **before** `left`, a stale occurrence that's already outside the window. Moving `left` there would move it *backwards*. Only jump if `last_seen[c] >= left`. With `"abba"`: at the final `a`, `last_seen['a'] = 0` but `left = 2`, so you must not move `left` back to 1.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(n)$. `right` visits each index once, and `left` only jumps forward. Each step is $O(1)$ dict work.
- **Space:** $O(\min(n, \Sigma))$, where $\Sigma$ is the alphabet size (at most 128 for ASCII). That's effectively $O(1)$ for a fixed charset.

### Code: index-jump version (optimal)

```python
from __future__ import annotations


class Solution:
    def lengthOfLongestSubstring(self, s: str) -> int:
        last_seen: dict[str, int] = {}  # char -> most recent index
        left = 0                         # window is s[left : right + 1]
        best = 0

        for right, ch in enumerate(s):
            # Duplicate INSIDE the current window -> jump left past its previous occurrence.
            # The `>= left` guard ignores stale indices from before the window ("abba").
            if ch in last_seen and last_seen[ch] >= left:
                left = last_seen[ch] + 1

            last_seen[ch] = right
            best = max(best, right - left + 1)

        return best


if __name__ == "__main__":
    s = Solution()
    assert s.lengthOfLongestSubstring("abcabcbb") == 3
    assert s.lengthOfLongestSubstring("bbbbb") == 1
    assert s.lengthOfLongestSubstring("pwwkew") == 3
    assert s.lengthOfLongestSubstring("") == 0
    assert s.lengthOfLongestSubstring(" ") == 1
    assert s.lengthOfLongestSubstring("abba") == 2          # stale-index trap
    assert s.lengthOfLongestSubstring("dvdf") == 3          # "vdf"
    assert s.lengthOfLongestSubstring("tmmzuxt") == 5       # "mzuxt"
    print("all tests passed")
```

### Code: template version (set + shrink loop)

This is less optimal by a constant factor but follows the generic template exactly. Learn it, because it generalizes to "at most K distinct" and similar problems where a jump target doesn't exist.

```python
def length_of_longest_substring(s: str) -> int:
    window: set[str] = set()
    left = best = 0
    for right, ch in enumerate(s):
        while ch in window:          # invalid -> shrink until the duplicate leaves
            window.remove(s[left])
            left += 1
        window.add(ch)
        best = max(best, right - left + 1)
    return best


assert length_of_longest_substring("abcabcbb") == 3
assert length_of_longest_substring("abba") == 2
```

### Dry run: `"abba"` (index-jump version)

| `right` | `ch` | `last_seen` before | Jump? | `left` | Window | `best` |
|---|---|---|---|---|---|---|
| 0 | a | {} | — | 0 | `a` | 1 |
| 1 | b | {a:0} | — | 0 | `ab` | 2 |
| 2 | b | {a:0, b:1} | 1 ≥ 0 ✓ → left=2 | 2 | `b` | 2 |
| 3 | a | {a:0, b:2} | 0 ≥ 2 ✗ (stale) | 2 | `ba` | 2 |

Without the guard, step 3 would set `left = 1`, giving the window `"bba"` and the wrong answer 3.

### Common mistakes

- Missing the `>= left` guard. It passes every sample test and fails on `"abba"`.
- Writing `left = max(left, last_seen[ch] + 1)` is an equivalent fix. Know both.
- Computing the length as `right - left` (off by one).

### Variants to try (optional)

- **LC 424 Longest Repeating Character Replacement:** valid when `window_len − max_freq ≤ k`.
- **LC 340 / 159 Longest Substring with At Most K / 2 Distinct Characters:** `Counter` state with shrink-while-`len(counter) > k`.
- **LC 76 Minimum Window Substring:** the "shortest valid" flip. Record inside the `while valid` loop.

</details>

---

[← Day 5 — asyncio](day-05-asyncio-event-loop.md) · [Day 7 — Consolidation →](day-07-consolidation.md)
