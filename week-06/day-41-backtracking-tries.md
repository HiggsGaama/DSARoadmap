# Day 41 — Backtracking & Tries: Enumerating a Search Tree, Pruning It Hard

[← Week 6](README.md) · **Track:** DSA · **Time box:** 60 min (pattern 15 · attempt 20 · walkthrough 15 · trie 10)

---

## 1. Algorithmic Pattern

**Backtracking** is DFS over an implicit **decision tree**. Each node is a partial solution, each edge is one choice, and each leaf is either a complete solution or a dead end. You build the solution incrementally and **undo** each choice after exploring it, so a single mutable `path` serves the whole search.

### The template

```text
def backtrack(start, path, remaining):
    if is_solution(remaining):
        results.append(path.copy())          # COPY: `path` keeps mutating after this
        return
    for i in range(start, len(choices)):
        if not feasible(choices[i], remaining):
            break / continue                 # PRUNE: cut whole subtrees early
        path.append(choices[i])              # choose
        backtrack(next_start(i), path, remaining - choices[i])   # explore
        path.pop()                           # un-choose
```

### The `start` index controls the shape of the output

| Want | Recursion passes | Example |
|---|---|---|
| **Combinations, each item once** | `i + 1` | Subsets (LC 78), Combination Sum II (LC 40) |
| **Combinations, items reusable** | `i` (stay on the same item) | **Combination Sum (LC 39)** |
| **Permutations** | `0`, plus a `used[]` set | Permutations (LC 46) |
| **Skip duplicate values** (sorted input) | `if i > start and a[i] == a[i-1]: continue` | Subsets II, Combination Sum II. Same idea as 3Sum's dedup (Day 4) |

Never iterating *before* `start` is what prevents `[2,3]` and `[3,2]` from both appearing.

### Pruning is where the performance comes from

Backtracking is exponential in the worst case. Pruning is how you make it practical:

- **Sort first, then `break`, not `continue`:** once `candidate > remaining`, every later candidate is bigger too.
- **Feasibility bounds:** "not enough characters left to finish", "already exceeded the best answer" (branch and bound).
- **Constraint propagation:** N-Queens with column and diagonal sets gives $O(1)$ conflict checks.

### Tries: shared-prefix search

A **trie** (prefix tree) stores strings character by character. Each node maps a character to a child, plus an `is_end` flag (or the full word). Insert and lookup cost $O(L)$ for a word of length $L$, *independent of how many words are stored*.

| Need | `set[str]` | Trie |
|---|---|---|
| Exact membership | ✅ $O(L)$ hash; less memory | ✅ $O(L)$ |
| "Does any word start with `pre`?" | ❌ scan all words | ✅ $O(\text{len(pre)})$ |
| Enumerate all completions of `pre` | ❌ scan all words | ✅ walk the subtree |
| Prune a search as soon as a prefix is impossible | ❌ | ✅ **the key use** |

That last row is where tries and backtracking meet. In **Word Search II (LC 212)**, you DFS a letter grid while walking a trie of the dictionary *at the same time*, and abandon a path the moment the current prefix isn't a trie node. Production relatives include autocomplete, routing tables (longest-prefix match), tokenizer vocabularies (BPE merges and prefix matching), and constrained decoding for LLMs, where a trie of allowed outputs masks invalid next tokens.

## 2. Identification Rule

Use backtracking when:

1. ✅ The problem says **"return all"**: all combinations, permutations, subsets, partitions, paths or valid boards.
2. ✅ The input is **small**: `n <= 20`, `target <= 40`, a 9×9 board. Small constraints are the setter telling you exponential is acceptable.
3. ✅ You build candidates **step by step**, and a partial candidate can be **rejected early** (constraint satisfaction: Sudoku, N-Queens, word search).

**Backtracking vs DP:** "*How many* ways / *minimum* coins?" → DP (Day 37), because you only need a count and overlapping subproblems make it polynomial. "*List* every way" → backtracking, because the output itself can be exponential and no DP avoids writing it out.

Add a **trie** when you're searching for many words or prefixes at once, and pruning by prefix would cut the search.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Combination Sum` | **NeetCode** | Decision-tree drawing that makes the "reuse vs advance" index obvious |
| `take U forward recursion playlist combination sum` | **take U forward** (Striver) | Recursion → backtracking series: subsets, permutations, combination sums, N-Queens |
| `NeetCode Implement Trie Prefix Tree` + `NeetCode Word Search II` | **NeetCode** | Trie basics, then trie-guided backtracking |

---

## 4. Daily Problem — LeetCode 39: Combination Sum (Medium)

**Statement:** Given an array of **distinct** integers `candidates` and an integer `target`, return all **unique combinations** of candidates that sum to `target`, in any order. The same number may be chosen **any number of times**. Two combinations are the same if they use the same multiset of numbers.

```text
Input:  candidates = [2, 3, 6, 7], target = 7
Output: [[2, 2, 3], [7]]

Input:  candidates = [2, 3, 5], target = 8
Output: [[2, 2, 2, 2], [2, 3, 3], [3, 5]]

Input:  candidates = [2], target = 1
Output: []
```

**Constraints:** `1 <= len(candidates) <= 30`, `2 <= candidates[i] <= 40` (distinct), `1 <= target <= 40`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) Draw the decision tree for `[2,3,6,7], 7`. What makes `[2,2,3]` and `[3,2,2]` appear separately? (2) What index should the recursive call start from if reuse is allowed?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **The decision at each node:** pick which candidate to add next. The node's state is `(start, path, remaining)`.
2. **Avoid permutation duplicates:** only consider candidates at index `>= start`. A combination is then generated in non-decreasing index order exactly once: `[2,2,3]` is produced, and `[3,2,2]` never is.
3. **Allow reuse:** recurse with `start = i`, not `i + 1`, so the same candidate can be picked again.
4. **Stop conditions:** `remaining == 0` means a solution, so record a **copy** of `path`. `remaining < 0` means a dead end.
5. **Prune:** sort the candidates. If `candidates[i] > remaining`, then every later candidate is too big as well, so `break` out of the loop instead of recursing into hopeless branches.

```text
                           [] r=7
          ┌────────────┬────┴──────┬────────┐
       +2 r=5      +3 r=4      +6 r=1    +7 r=0 ✓[7]
     ┌───┼────┐     ┌──┴──┐       (6,7 > 1: break)
  +2 r=3 +3 r=2 …  +3 r=1 …
  ┌──┴──┐  (3>2: break)  (3>1: break)
+2 r=1 +3 r=0 ✓[2,2,3]
(2>1… ✗)
```

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

Let $T$ = `target`, $M$ = the smallest candidate, and $N$ = the number of candidates.

- **Time:** $O(N^{T/M + 1})$ as a loose upper bound. The tree depth is at most $T/M$ (all smallest coins), and each node branches at most $N$ ways. Sorting plus `break` prunes heavily in practice. The output itself can be exponential, so no algorithm beats output size.
- **Space:** $O(T/M)$ recursion depth and `path` length, excluding the output.

### Code

```python
from __future__ import annotations


class Solution:
    def combinationSum(self, candidates: list[int], target: int) -> list[list[int]]:
        nums = sorted(candidates)            # enables `break` pruning
        results: list[list[int]] = []
        path: list[int] = []

        def backtrack(start: int, remaining: int) -> None:
            if remaining == 0:
                results.append(path.copy())  # copy: `path` is reused by the whole search
                return
            for i in range(start, len(nums)):
                if nums[i] > remaining:
                    break                    # sorted -> all later candidates are too big
                path.append(nums[i])         # choose
                backtrack(i, remaining - nums[i])   # explore; `i` (not i+1) allows reuse
                path.pop()                   # un-choose

        backtrack(0, target)
        return results


# ---------------------------------------------------------------------------
# Trie mini-section: the structure behind prefix pruning (LC 208 / LC 212)
# ---------------------------------------------------------------------------
class TrieNode:
    __slots__ = ("children", "is_end")

    def __init__(self) -> None:
        self.children: dict[str, TrieNode] = {}
        self.is_end = False


class Trie:
    def __init__(self) -> None:
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def _walk(self, prefix: str) -> TrieNode | None:
        node = self.root
        for ch in prefix:
            if (node := node.children.get(ch)) is None:
                return None
        return node

    def search(self, word: str) -> bool:
        node = self._walk(word)
        return node is not None and node.is_end

    def starts_with(self, prefix: str) -> bool:
        return self._walk(prefix) is not None

    def completions(self, prefix: str, limit: int = 10) -> list[str]:
        """Autocomplete: DFS the subtree under `prefix` (itself a tiny backtracking search)."""
        start = self._walk(prefix)
        out: list[str] = []
        chars: list[str] = list(prefix)

        def dfs(node: TrieNode) -> None:
            if len(out) >= limit:
                return                       # prune once we have enough
            if node.is_end:
                out.append("".join(chars))
            for ch in sorted(node.children):
                chars.append(ch)
                dfs(node.children[ch])
                chars.pop()

        if start is not None:
            dfs(start)
        return out


if __name__ == "__main__":
    s = Solution()
    assert s.combinationSum([2, 3, 6, 7], 7) == [[2, 2, 3], [7]]
    assert s.combinationSum([2, 3, 5], 8) == [[2, 2, 2, 2], [2, 3, 3], [3, 5]]
    assert s.combinationSum([2], 1) == []
    assert s.combinationSum([7, 3, 2], 18) == s.combinationSum([2, 3, 7], 18)  # order-independent
    assert len(s.combinationSum([2, 3, 5], 40)) == 34   # brute-force verified
    assert all(sum(c) == 40 for c in s.combinationSum([2, 3, 5], 40))

    trie = Trie()
    for w in ["embed", "embedding", "embeddings", "emit", "error", "eval"]:
        trie.insert(w)
    assert trie.search("embed") and not trie.search("emb")
    assert trie.starts_with("emb") and not trie.starts_with("xyz")
    assert trie.completions("emb") == ["embed", "embedding", "embeddings"]
    assert trie.completions("e", limit=2) == ["embed", "embedding"]
    print("all tests passed")
```

### Dry run: `[2, 3, 6, 7]`, target 7 (sorted already)

| Call `(start, remaining)` | `path` | Action |
|---|---|---|
| (0, 7) | [] | try 2 |
| (0, 5) | [2] | try 2 |
| (0, 3) | [2,2] | try 2 → (0,1): 2 > 1 → break; pop. Try 3 → (1,0) ✓ record `[2,2,3]` |
| (0, 3) | [2,2] | try 6: 6 > 3 → **break** |
| (0, 5) | [2] | try 3 → (1,2): 3 > 2 break. Try 6: 6 > 5 → **break** |
| (0, 7) | [] | try 3 → (1,4): 3 → (1,1): break … no solution |
| (0, 7) | [] | try 6 → (2,1): 6 > 1 break. Try 7 → (3,0) ✓ record `[7]` |

### Common mistakes

- `results.append(path)` without `.copy()` leaves every result pointing at the same (eventually empty) list. That's Day 1 aliasing again.
- Recursing with `i + 1` silently forbids reuse and misses `[2,2,3]`. Recursing with `0` produces permutations (`[2,3,2]`, `[3,2,2]`, …).
- Using `continue` instead of `break` after sorting is correct but wastes work.
- Forgetting `path.pop()` corrupts every sibling branch.

### Variants to try (optional)

- **LC 40 Combination Sum II:** each item used once, input has duplicates. Use `i + 1` plus the `i > start` dedup.
- **LC 78 / 90 Subsets I / II**, **LC 46 Permutations**, **LC 51 N-Queens**.
- **LC 208 Implement Trie**, then **LC 212 Word Search II**: DFS the grid with the trie as a live prefix filter, and remove found words from the trie to prune further.

</details>

---

[← Day 40 — Streaming LLM Responses](day-40-streaming-llm-responses.md) · [Day 42 — Final Consolidation →](day-42-final-consolidation.md)
