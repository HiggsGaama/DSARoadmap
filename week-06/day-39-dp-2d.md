# Day 39 — 2-D Dynamic Programming: Two Sequences, One Table

[← Week 6](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 15 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

When a problem compares or combines **two sequences** (or walks a grid), the natural state needs **two indices**: `dp[i][j]` = the answer for the first `i` items of A and the first `j` items of B. The table has $(m+1) \times (n+1)$ cells, where row 0 and column 0 represent empty prefixes, and each cell depends on a few **neighbouring** cells.

### The canonical shape: match / skip

```text
            if A[i-1] == B[j-1]:  dp[i][j] = dp[i-1][j-1] + (gain)        # use both chars together
            else:                 dp[i][j] = best(dp[i-1][j], dp[i][j-1]) # drop a char from A or from B
```

| Problem | Cell meaning | On match | On mismatch |
|---|---|---|---|
| **LCS (LC 1143)** | length of LCS of `A[:i]`, `B[:j]` | `diag + 1` | `max(up, left)` |
| Edit Distance (LC 72) | min edits `A[:i] → B[:j]` | `diag` | `1 + min(up, left, diag)` |
| Distinct Subsequences (LC 115) | # ways `B[:j]` appears in `A[:i]` | `up + diag` | `up` |
| Unique Paths (LC 62), grid | # paths to `(i, j)` | — | `up + left` |

### Fill order = dependency order

Each cell reads **up** `(i−1, j)`, **left** `(i, j−1)` and **diagonal** `(i−1, j−1)`. Filling row by row, left to right, guarantees all three are computed before you need them. Getting the order wrong is the most common 2-D DP bug: you end up reading zeros that haven't been computed.

### Space optimization: rolling rows

Row `i` reads only from row `i−1` and from itself, so two rows are enough: $O(n)$ space. With **one** row you need to save the diagonal before overwriting it:

```text
prev_diag = dp[j]           # this is old row's dp[j-1] for the NEXT column; save before overwrite
```

Put the **shorter** string on the column axis to get $O(\min(m, n))$ space. The trade-off is that you can no longer reconstruct the actual subsequence, because that needs the full table.

## 2. Identification Rule

Think 2-D DP when:

1. ✅ There are **two strings or arrays**, and the question asks about **subsequences, alignment, edits, interleaving or matching**: *common subsequence*, *edit distance*, *transform A into B*, *is S an interleaving of A and B*, *wildcard / regex match*.
2. ✅ It's a **grid** with movement restricted to right/down (or similar), asking for a count, min cost or max value.
3. ✅ One sequence with an **interval** state, `dp[i][j]` over a substring `s[i..j]`: Longest Palindromic Subsequence, burst balloons, matrix-chain multiplication.

**Sanity check:** if the problem asks about *contiguous* substrings of a single string, try a sliding window (Day 6) first. Subsequence plus two inputs is the 2-D DP signal.

**Production relevance:** `difflib`, `git diff` (Myers' algorithm is an optimized LCS/edit-script search), fuzzy matching of entity names, and evaluation metrics like ROUGE-L for LLM summaries are all LCS-based.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Longest Common Subsequence` | **NeetCode** | Table walk-through with the bottom-up fill |
| `take U forward DP on strings longest common subsequence` | **take U forward** (Striver) | DP-on-strings series: LCS → printing the LCS → edit distance → wildcard |
| `NeetCode Edit Distance` | **NeetCode** | The natural follow-up: same table, three transitions |

---

## 4. Daily Problem — LeetCode 1143: Longest Common Subsequence (Medium)

**Statement:** Given two strings `text1` and `text2`, return the length of their **longest common subsequence**, or `0` if there is none. A subsequence keeps the relative order of characters but may skip characters. `"ace"` is a subsequence of `"abcde"`; `"aec"` is not.

```text
Input:  text1 = "abcde", text2 = "ace"
Output: 3          # "ace"

Input:  text1 = "abc", text2 = "abc"
Output: 3

Input:  text1 = "abc", text2 = "def"
Output: 0
```

**Constraints:** `1 <= len(text1), len(text2) <= 1000`; lowercase English letters only.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) Compare the *last* characters of both prefixes. If they're equal, what can you safely do? (2) If they're different, one of the two can't be the last character of the LCS. Which subproblems does that leave?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **Brute force:** generate all $2^m$ subsequences of `text1` and test each one against `text2`. That's exponential.
2. **Look at the last characters** of `A[:i]` and `B[:j]`:
   - **Equal:** some LCS uses both. If an optimal LCS didn't, you could append this shared character and make it longer. So `dp[i][j] = dp[i−1][j−1] + 1`.
   - **Different:** they can't *both* end the LCS. Either drop `A`'s last character or drop `B`'s: `dp[i][j] = max(dp[i−1][j], dp[i][j−1])`.
3. **Base case:** an empty prefix has LCS 0, so row 0 and column 0 are all zeros. That's why the table is `(m+1) × (n+1)`, which avoids `i−1 = −1` edge checks.
4. **Fill order:** row by row, left to right, because every cell needs up, left and diagonal.
5. **Space:** each row needs only the previous row, so roll it into one array plus a saved diagonal.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(m \cdot n)$. Each cell takes $O(1)$ work, so 1000 × 1000 = $10^6$ cells.
- **Space:** $O(m \cdot n)$ for the full table (needed if you want to reconstruct the string), or $O(\min(m, n))$ with one rolling row.

### Code

```python
from __future__ import annotations


class Solution:
    def longestCommonSubsequence(self, text1: str, text2: str) -> int:
        """Full table: easiest to reason about; supports reconstruction."""
        m, n = len(text1), len(text2)
        # dp[i][j] = LCS length of text1[:i] and text2[:j]; row/col 0 = empty prefix.
        dp = [[0] * (n + 1) for _ in range(m + 1)]  # NOT [[0]*(n+1)]*(m+1): that aliases rows

        for i in range(1, m + 1):
            for j in range(1, n + 1):
                if text1[i - 1] == text2[j - 1]:
                    dp[i][j] = dp[i - 1][j - 1] + 1             # use both characters
                else:
                    dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])  # drop one of them
        return dp[m][n]


def lcs_one_row(a: str, b: str) -> int:
    """O(min(m, n)) space: one row, plus the saved diagonal."""
    if len(b) > len(a):
        a, b = b, a                       # keep the shorter string on the column axis
    row = [0] * (len(b) + 1)
    for ch_a in a:
        diag = 0                          # dp[i-1][0] is always 0
        for j, ch_b in enumerate(b, start=1):
            up = row[j]                   # dp[i-1][j] before we overwrite it
            row[j] = diag + 1 if ch_a == ch_b else max(up, row[j - 1])
            diag = up                     # becomes dp[i-1][j-1] for the next column
    return row[-1]


def lcs_string(a: str, b: str) -> str:
    """Reconstruct one LCS by walking the full table back from the corner."""
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            dp[i][j] = dp[i - 1][j - 1] + 1 if a[i - 1] == b[j - 1] else max(dp[i - 1][j], dp[i][j - 1])
    out: list[str] = []
    i, j = m, n
    while i and j:
        if a[i - 1] == b[j - 1]:
            out.append(a[i - 1])
            i, j = i - 1, j - 1
        elif dp[i - 1][j] >= dp[i][j - 1]:
            i -= 1
        else:
            j -= 1
    return "".join(reversed(out))


if __name__ == "__main__":
    s = Solution()
    cases = [("abcde", "ace", 3), ("abc", "abc", 3), ("abc", "def", 0),
             ("bsbininm", "jmjkbkjkv", 1), ("oxcpqrsvwf", "shmtulqrypy", 2), ("a", "a", 1)]
    for a, b, want in cases:
        assert s.longestCommonSubsequence(a, b) == want, (a, b)
        assert lcs_one_row(a, b) == want, (a, b)
        assert len(lcs_string(a, b)) == want, (a, b)
    assert lcs_string("abcde", "ace") == "ace"
    print("all tests passed")
```

### Dry run: `text1 = "abcde"` (rows), `text2 = "ace"` (columns)

|   | "" | a | c | e |
|---|---|---|---|---|
| "" | 0 | 0 | 0 | 0 |
| a | 0 | **1** ↖ | 1 | 1 |
| b | 0 | 1 | 1 | 1 |
| c | 0 | 1 | **2** ↖ | 2 |
| d | 0 | 1 | 2 | 2 |
| e | 0 | 1 | 2 | **3** ↖ |

↖ marks a match (diagonal + 1). Every other cell is `max(up, left)`. Following the ↖ cells back from the bottom-right corner spells out `"ace"`.

### Common mistakes

- **`[[0] * (n+1)] * (m+1)`** creates `m+1` references to *one* list, so writing a cell writes the whole column. This is the classic Python 2-D DP bug, and it's Day 1 again: names are references.
- Off-by-one: with padding, `dp[i][j]` refers to `text1[i-1]` and `text2[j-1]`.
- In the one-row version, forgetting to save `up` before overwriting it, which makes `diag` wrong.
- Confusing *subsequence* with *substring*. For the Longest Common **Substring**, set the cell to `0` on a mismatch and track the global max.

### Variants to try (optional)

- **LC 72 Edit Distance:** the canonical 2-D DP interview problem.
- **LC 583 Delete Operation for Two Strings:** `m + n − 2·LCS`.
- **LC 1092 Shortest Common Supersequence:** reconstruct using the LCS table.
- **LC 62 / 64 Unique Paths, Minimum Path Sum:** grid versions with the same fill order and rolling-row trick.

</details>

---

[← Day 38 — Resilience Patterns](day-38-resilience-patterns.md) · [Day 40 — Streaming LLM Responses →](day-40-streaming-llm-responses.md)
