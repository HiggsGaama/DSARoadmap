# Day 37 — 1-D Dynamic Programming: State, Transition, Base Case

[← Week 6](README.md) · **Track:** DSA · **Time box:** 55 min (pattern 15 · attempt 20 · walkthrough 15 · variants 5)

---

## 1. Algorithmic Pattern

Dynamic programming works when a problem has two properties:

1. **Optimal substructure:** the best answer for size `i` is built from best answers to smaller sizes.
2. **Overlapping subproblems:** naive recursion solves the same smaller instances over and over, often exponentially many times.

DP solves each subproblem **once** and stores the result. "1-D" means the state is a single integer: an index, a remaining amount or a length.

### The four-question recipe (answer these on paper *before* coding)

| Question | Coin Change answer |
|---|---|
| **State:** what does `dp[i]` mean, in words? | `dp[a]` = fewest coins that sum to exactly `a` |
| **Transition:** how does `dp[i]` come from smaller states? | `dp[a] = 1 + min(dp[a − c] for c in coins if c <= a)` |
| **Base case:** what's known without recursion? | `dp[0] = 0`; unreachable = `∞` |
| **Order / answer:** which way do you fill, and which cell is the answer? | `a = 1 .. amount`, answer `dp[amount]` (or −1 if `∞`) |

If you can't write the state as one precise English sentence, you aren't ready to code.

### Two implementations of the same recurrence

| | Top-down (memoized recursion) | Bottom-up (tabulation) |
|---|---|---|
| How | Recursive function + `@cache` | Loop that fills an array in dependency order |
| Pros | Closest to the recurrence; computes only the states it needs | No recursion limit, no call overhead, easy to space-optimize |
| Cons | Python's recursion limit (~1000 by default) and function-call overhead | You must figure out a valid fill order |
| Use when | Prototyping, or a sparse or irregular state space | Production code and large inputs |

### Space optimization

If `dp[i]` depends only on the last *k* cells, keep only *k* variables. For example, Climbing Stairs and House Robber need two variables, giving $O(1)$ space. Coin Change depends on `dp[a − c]` for arbitrary coin values, so it needs the full array.

### Families of 1-D DP

- **Linear sequence decisions:** Climbing Stairs, House Robber, Decode Ways. `dp[i]` depends on `dp[i−1]` and `dp[i−2]`.
- **Unbounded knapsack / "make amount":** Coin Change, Coin Change II (count ways), Perfect Squares. `dp[a]` depends on `dp[a − item]`.
- **Ending-at-i:** Maximum Subarray (Kadane), Longest Increasing Subsequence (Day 42). `dp[i]` = best answer *that ends exactly at i*, and the overall answer is `max(dp)`.

## 2. Identification Rule

Think DP when **all** of these hold:

1. ✅ The question asks for **min / max / count of ways / is it possible**, not for the actual list of all solutions. (Listing all solutions is backtracking, Day 41.)
2. ✅ You make a **sequence of choices** (take a coin, rob or skip a house, split here or not), and each choice reduces the problem to a smaller instance of the same problem.
3. ✅ A naive recursion tree has **repeated nodes**. Draw two levels and look for the same argument appearing twice.

**Strong signals:** "minimum number of", "number of distinct ways", "can you reach", unlimited reuse of items ("infinite supply"), or constraints like `amount <= 10^4` (small enough to tabulate).

**Anti-signals:**

- ❌ Greedy is **provably** correct: interval scheduling (Week 4) or coin systems like US coins. Coin Change exists precisely because greedy fails for arbitrary denominations: coins `[1, 3, 4]`, amount 6: greedy gives 4+1+1 (3 coins), optimal is 3+3 (2 coins).
- ❌ You need to *enumerate* every solution → backtracking.
- ❌ The state space is huge, e.g. `amount <= 10^9` → look for math, BFS on a smaller graph, or another formulation.

---

## 3. Curated Video Resource

| Search on YouTube | Channel | Notes |
|---|---|---|
| `NeetCode Coin Change` | **NeetCode** | Draws the recursion tree → memo → bottom-up table |
| `take U forward DP series` (Striver's DP playlist) | **take U forward** | The most systematic DP course on YouTube: recursion → memo → tabulation → space optimization for every problem |
| `MIT 6.006 dynamic programming Erik Demaine` | **MIT OpenCourseWare** | The "SRTBOT" framework (subproblem, relation, topological order, base, original, time). Deep, but worth one lecture |

---

## 4. Daily Problem — LeetCode 322: Coin Change (Medium)

**Statement:** You're given an integer array `coins` of distinct denominations and an integer `amount`. Return the **fewest coins** needed to make up exactly `amount`. If it can't be made, return `-1`. You have an unlimited number of each coin.

```text
Input:  coins = [1, 2, 5], amount = 11
Output: 3            # 5 + 5 + 1

Input:  coins = [2], amount = 3
Output: -1

Input:  coins = [1], amount = 0
Output: 0
```

**Constraints:** `1 <= len(coins) <= 12`, `1 <= coins[i] <= 2^31 − 1`, `0 <= amount <= 10^4`.

> ⏱ Attempt it for 20 minutes. Hint ladder: (1) Write the recursion `f(a)` = fewest coins for amount `a`. What's the last coin you add? (2) Draw `f(11)` with coins `[1,2,5]` two levels deep. Do you see `f(9)` twice?

<details markdown="1">
<summary><strong>Intuition (open first if stuck)</strong></summary>

1. **Think about the last coin.** Any optimal way to make `a` ends with some coin `c`. What's left, `a − c`, must itself be made optimally (optimal substructure). So `f(a) = 1 + min over c of f(a − c)`.
2. **Base cases:** `f(0) = 0`, and `f(negative)` is impossible. Represent "impossible" as `∞` so `min` ignores it automatically.
3. **Overlap:** `f(11)` → `f(10)`, `f(9)`, `f(6)`. Then `f(10)` → `f(9)`, `f(8)`, `f(5)`, so `f(9)` is computed twice already, and it gets exponentially worse. Memoizing collapses this to `amount + 1` distinct states.
4. **Bottom-up order:** `f(a)` depends only on smaller amounts, so filling `a = 1, 2, …, amount` guarantees the dependencies are ready.
5. **Why greedy fails:** "always take the largest coin" commits early. DP considers every last coin at every amount.

</details>

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Complexity

- **Time:** $O(A \cdot C)$, where $A$ = `amount` and $C$ = number of coins: each of the $A$ cells tries every coin.
- **Space:** $O(A)$ for the table. It can't be reduced to $O(1)$, because the dependencies reach back by arbitrary coin values.

### Code

```python
from __future__ import annotations

from functools import cache
from math import inf


class Solution:
    def coinChange(self, coins: list[int], amount: int) -> int:
        # dp[a] = fewest coins summing to exactly a; inf = unreachable.
        dp: list[float] = [0.0] + [inf] * amount

        for a in range(1, amount + 1):
            for c in coins:
                # Transition: last coin is c; the rest (a - c) must be optimal.
                if c <= a and dp[a - c] + 1 < dp[a]:
                    dp[a] = dp[a - c] + 1

        return int(dp[amount]) if dp[amount] != inf else -1


def coin_change_top_down(coins: list[int], amount: int) -> int:
    """Same recurrence, memoized. Fine for interviews; recursion depth ~ amount / min(coins),
    so amount=10^4 with coin 1 would exceed the default recursion limit (1000)."""

    @cache
    def f(a: int) -> float:
        if a == 0:
            return 0
        if a < 0:
            return inf
        return 1 + min(f(a - c) for c in coins)

    best = f(amount)
    return int(best) if best != inf else -1


if __name__ == "__main__":
    s = Solution()
    assert s.coinChange([1, 2, 5], 11) == 3
    assert s.coinChange([2], 3) == -1
    assert s.coinChange([1], 0) == 0
    assert s.coinChange([1, 3, 4], 6) == 2            # greedy would give 3
    assert s.coinChange([186, 419, 83, 408], 6249) == 20
    assert s.coinChange([2, 5, 10, 1], 27) == 4
    assert s.coinChange([5], 10_000) == 2_000         # large table, fine bottom-up
    for coins, amount in ([1, 2, 5], 11), ([2], 3), ([1, 3, 4], 6), ([3, 7], 50):
        assert coin_change_top_down(coins, amount) == s.coinChange(coins, amount)
    print("all tests passed")
```

### Dry run: `coins = [1, 2, 5]`, `amount = 11`

| `a` | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `dp[a]` | 0 | 1 | 1 | 2 | 2 | 1 | 2 | 2 | 3 | 3 | 2 | **3** |
| best last coin | — | 1 | 2 | 1/2 | 2 | 5 | 1/5 | 2/5 | 1/2/5 | 2/5 | 5 | 1/5 |

`dp[11] = 1 + min(dp[10], dp[9], dp[6]) = 1 + min(2, 3, 2) = 3`.

### Common mistakes

- Initializing unreachable cells as `0` or `-1`. `min()` then picks them. Use `inf` (or `amount + 1`, which is larger than any real answer).
- Returning `dp[amount]` without mapping `inf` → `-1`.
- Greedy-first thinking. Always test `[1, 3, 4], 6`.
- Confusing it with **Coin Change II** (LC 518, *count* combinations). There the loop order matters: coins in the **outer** loop counts combinations, while amounts in the outer loop counts permutations.

### Variants to try (optional)

- **LC 70 Climbing Stairs / LC 198 House Robber:** the $O(1)$-space two-variable version.
- **LC 518 Coin Change II:** count ways; experiment with swapping the loops.
- **LC 279 Perfect Squares:** Coin Change where the "coins" are squares.
- **Reconstruct the coins:** store `choice[a] = c` alongside `dp[a]`, then walk back from `amount`.

</details>

---

[← Day 36 — Observability](day-36-observability-contextvars-otel.md) · [Day 38 — Resilience Patterns →](day-38-resilience-patterns.md)
