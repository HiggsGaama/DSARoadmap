# Day 7 — Consolidation & Self-Audit

[← Week 1](README.md) · **Track:** Audit · **Time box:** 60 min

| Block | Minutes | Purpose |
|---|---|---|
| A. Retrieval quiz | 12 | Recall without notes. This is what moves knowledge into long-term memory |
| B. Pattern recognition drill | 8 | Map problem statements to patterns in under 30 s each |
| C. Production code review | 15 | Find this week's gotchas in realistic code |
| D. Transfer problem (timed) | 20 | A new problem with a known pattern, no hints |
| E. Self-audit & gap log | 5 | Decide what to revisit |

**Rules:** write your answer *before* opening each collapsible. A wrong answer you wrote down teaches more than a right answer you only recognized.

---

## A. Retrieval Quiz (12 min)

**Q1.** Why does `sys.getrefcount(None)` return `4294967295` on 3.12+, and why does that matter for prefork servers?

<details markdown="1"><summary>Answer</summary>

PEP 683 made `None` (along with `True`, `False`, small ints and many interned strings) **immortal**: its refcount is a fixed sentinel and incref/decref are no-ops. With normal objects, every read changes the refcount, which writes to the page and breaks copy-on-write sharing after `fork()`. Immortal objects never dirty their pages. For your own long-lived objects, the equivalent tool is `gc.freeze()` before forking.

</details>

**Q2.** Your service's `tracemalloc` total returns to baseline after a large batch job, but container RSS stays 1.5 GB higher. Is it a leak? What do you do?

<details markdown="1"><summary>Answer</summary>

It's almost certainly **not a Python-level leak**. It's pymalloc fragmentation or a high-water mark: arenas are only returned to the OS when completely empty, and a few long-lived small objects can pin them. Fix it by streaming or batching instead of materializing, and by recycling workers (`--max-requests` + jitter, `--max-tasks-per-child`). If RSS *keeps growing* while the Python heap stays flat, suspect a native leak and use `memray --native`.

</details>

**Q3.** What creates a reference cycle in `except Exception as e: self.last_error = e`, and what does it keep alive?

<details markdown="1"><summary>Answer</summary>

`e.__traceback__` → traceback → frames → **every local variable in every frame** of the stack at raise time (request bodies, DB rows, tensors). Python deletes `e` at the end of the `except` block to break the frame↔exception cycle, but storing it on `self` keeps the whole graph alive. Store `repr(e)` or `traceback.format_exception(e)` instead.

</details>

**Q4.** A generator is in `GEN_SUSPENDED`. What exactly does `g.close()` do, and what happens if the generator responds to it with another `yield`?

<details markdown="1"><summary>Answer</summary>

It raises `GeneratorExit` at the paused `yield`, so `finally` and `__exit__` blocks run. If the generator returns or lets the exception propagate, the state becomes `GEN_CLOSED`. If it yields a value instead, Python raises `RuntimeError("generator ignored GeneratorExit")`.

</details>

**Q5.** Name two situations where a generator's `finally` doesn't run promptly on CPython after the consumer `break`s.

<details markdown="1"><summary>Answer</summary>

Any two of: (1) another reference to the generator survives (stored on an object or in a list, or captured in a traceback); (2) the generator is part of a reference cycle, so it waits for the cyclic GC; (3) it's an **async generator**, where finalization is always deferred to a scheduled `aclose()`; (4) a non-refcounting runtime such as PyPy. The fix is `contextlib.closing` / `aclosing`, or keeping resource ownership outside the generator.

</details>

**Q6.** Explain how `await reader.read()`, three coroutines deep, ends up parking the Task.

<details markdown="1"><summary>Answer</summary>

Each `await` delegates downward like `yield from`. At the bottom, the pending `Future`'s `__await__` does `yield self`. That object passes up through every suspended coroutine frame to `Task.__step`, which called `coro.send(None)`. The task registers `__wakeup` as the future's done-callback and returns control to the loop. When the future resolves, `__wakeup` → `__step` → `send(None)` resumes the whole chain at the bottom `yield`.

</details>

**Q7.** Why can `asyncio.create_task(work())`, with its return value discarded, fail to complete?

<details markdown="1"><summary>Answer</summary>

The event loop holds only **weak** references to tasks. If nothing else strongly references the task (or the future it's waiting on), it can be garbage-collected while suspended. Its exceptions also go unretrieved. Keep a strong reference (a set plus `add_done_callback(set.discard)`) or use a `TaskGroup`.

</details>

**Q8.** Does `await some_coroutine()` always give control back to the event loop?

<details markdown="1"><summary>Answer</summary>

No. Control returns only when something at the bottom of the await chain yields a pending future. A coroutine that returns without suspending, such as a cache hit, runs synchronously. A loop of such awaits can starve the event loop, and `await asyncio.sleep(0)` is the explicit way to yield.

</details>

**Q9.** In Longest Consecutive Sequence, why iterate over `num_set` instead of `nums`?

<details markdown="1"><summary>Answer</summary>

Duplicates. If the start of a long run appears many times in `nums`, each copy passes the `x−1 not in set` check and re-walks the entire run, which degrades to $O(n^2)$. The set contains each start once.

</details>

**Q10.** State the property a constraint must have for a variable sliding window to be correct, and give a constraint that lacks it.

<details markdown="1"><summary>Answer</summary>

**Monotonicity:** if a window is invalid, every window containing it is also invalid, so advancing `left` never discards a future answer. Counter-example: "subarray sum equals k" with negative numbers, where extending the window can bring the sum back down to k. Use prefix sums + hash map for that.

</details>

---

## B. Pattern Recognition Drill (8 min)

For each statement, name the pattern and the **one phrase** that gave it away. Aim for under 30 seconds each.

1. "Given a sorted array, return the number of pairs whose sum is less than `target`."
2. "Return the length of the shortest contiguous subarray whose sum ≥ `s`. All numbers are positive."
3. "Given a list of email strings, group them by their normalized domain."
4. "Determine whether a string is a palindrome after removing non-alphanumeric characters, using O(1) extra space."
5. "Return the indices of two numbers in an unsorted array that add up to `target`."
6. "Count subarrays whose sum equals `k`. The array may contain negative numbers."
7. "Find all start indices of `p`'s anagrams in `s`."
8. "Remove duplicates from a sorted array in place and return the new length."

<details markdown="1"><summary>Answers</summary>

| # | Pattern | Giveaway |
|---|---|---|
| 1 | Two Pointers (converging). When `a[lo]+a[hi] < target`, add `hi − lo` pairs at once | "sorted", "pairs" |
| 2 | Sliding Window, variable size, "shortest valid" flip | "contiguous", "shortest", "positive" (monotone) |
| 3 | Hashing: grouping by canonical key | "group by" |
| 4 | Two Pointers (converging, skipping invalid chars) | "palindrome", "O(1) space" |
| 5 | Hashing: complement map | "indices", "unsorted". Sorting would destroy the indices |
| 6 | Prefix sums + hash map (**not** a sliding window) | "negative numbers" breaks monotonicity |
| 7 | Sliding Window, fixed size `len(p)`, with a `Counter` | "anagram", "start indices" in `s` |
| 8 | Two Pointers, reader/writer | "in place", "sorted" |

</details>

---

## C. Production Code Review (15 min)

This handler "works" in staging. In production its memory climbs steadily, p99 latency spikes under load, some audit events go missing, and the processed-count metric is always zero. **Find at least six defects** tied to this week's material, then write the fixed version before opening the answer.

```python
import asyncio
import json
from functools import lru_cache

import requests


class FeatureClient:
    def __init__(self, base_url: str) -> None:
        self.base_url = base_url
        self.session = requests.Session()

    @lru_cache(maxsize=None)
    def get_features(self, user_id: str) -> dict:
        return self.session.get(f"{self.base_url}/users/{user_id}").json()


def read_events(path: str):
    f = open(path)
    for line in f:
        yield json.loads(line)
    f.close()


async def emit_audit(features: dict) -> None:
    ...


async def handle_batch(base_url: str, path: str) -> int:
    client = FeatureClient(base_url)          # new client per batch
    events = read_events(path)
    print(f"received {sum(1 for _ in events)} events")

    processed = 0
    for event in events:
        features = client.get_features(event["user_id"])
        asyncio.create_task(emit_audit(features))
        if features.get("kill_switch"):
            break
        processed += 1
    return processed
```

<details markdown="1"><summary>Defects & fixed version</summary>

| # | Defect | Consequence | Day |
|---|---|---|---|
| 1 | `@lru_cache(maxsize=None)` on a method | Cache keys include `self`, so every per-batch `FeatureClient` (and its `Session`, sockets and pool) is retained forever. Unbounded growth | 1 |
| 2 | `f.close()` not in `finally`, no `with` | The file descriptor leaks on exception **or** on early exit (`break`) | 3 |
| 3 | Generator consumed by `sum(...)` before the loop | The loop sees zero events, so `processed` is always 0, silently | 3 |
| 4 | `break` leaves `read_events` suspended | Even with a `finally`, cleanup waits for GC. Use `closing()` | 3 |
| 5 | Sync `requests` inside `async def` | Blocks the event loop for every HTTP call, stalling all concurrent requests (p99 spikes) | 5 |
| 6 | `asyncio.create_task` result discarded | Tasks can be GC'd mid-flight (missing audits), and their exceptions are never retrieved | 5 |
| 7 | No timeout on the HTTP call | A slow dependency hangs the worker indefinitely | 5 |
| 8 | Caching a mutable `dict` result | Callers mutating the returned dict corrupt the cache for everyone | bonus |

```python
from __future__ import annotations

import asyncio
import json
from collections.abc import Iterator
from contextlib import closing
from typing import Any

import httpx

_background: set[asyncio.Task[None]] = set()


def read_events(path: str) -> Iterator[dict[str, Any]]:
    with open(path, encoding="utf-8") as f:        # cleanup on exhaustion, error, or close()
        for line in f:
            yield json.loads(line)


async def emit_audit(features: dict[str, Any]) -> None: ...


def _spawn(coro: Any) -> None:
    task = asyncio.create_task(coro)
    _background.add(task)
    task.add_done_callback(_background.discard)


async def handle_batch(client: httpx.AsyncClient, path: str) -> int:
    # Client injected and shared (connection pool reuse); cache is a plain per-batch dict.
    cache: dict[str, dict[str, Any]] = {}
    processed = 0
    events = read_events(path)
    with closing(events):                            # deterministic cleanup on `break`
        for event in events:
            uid = event["user_id"]
            if uid not in cache:
                resp = await client.get(f"/users/{uid}", timeout=2.0)
                resp.raise_for_status()
                cache[uid] = resp.json()
            features = dict(cache[uid])              # defensive copy
            _spawn(emit_audit(features))
            if features.get("kill_switch"):
                break
            processed += 1
    print(f"processed {processed} events")           # count while iterating, not before
    return processed
```

Note: the sequential `await` per user is now non-blocking but still serial. Bounded *concurrent* fetching with a semaphore and `TaskGroup` is the Day 5 pattern, and it's the natural next refactor.

</details>

---

## D. Transfer Problem — LeetCode 11: Container With Most Water (Medium) · 20 min timed

No hints this time. Identify the pattern yourself.

**Statement:** You're given an array `height` of length `n`. The two endpoints of line `i` are `(i, 0)` and `(i, height[i])`. Find two lines that, together with the x-axis, form a container holding the most water. Return the maximum amount of water. The container can't be slanted.

```text
Input:  height = [1, 8, 6, 2, 5, 4, 8, 3, 7]
Output: 49            # lines at index 1 (h=8) and 8 (h=7): min(8,7) * (8-1) = 49

Input:  height = [1, 1]
Output: 1
```

**Constraints:** `2 <= n <= 10^5`, `0 <= height[i] <= 10^4`.

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Pattern

**Two Pointers (converging).** This is the Day 4 elimination argument applied to an area instead of a sum. The array *isn't* sorted, and that's what makes this a good transfer test.

### Intuition

1. `area(i, j) = min(h[i], h[j]) × (j − i)`. Brute force tries all pairs: $O(n^2)$.
2. Start with the widest container, `lo = 0`, `hi = n − 1`.
3. Suppose `h[lo] < h[hi]`. Every container that uses `lo` with some `hi' < hi` is **narrower**, and its height is still capped at `h[lo]` (the minimum can't exceed `h[lo]`). So none of them beats the current area. You can discard `lo` safely: `lo += 1`.
4. Symmetrically, if `h[hi] <= h[lo]`, discard `hi`.
5. Each step eliminates one line, giving at most `n − 1` steps.

**Why moving the taller side is pointless:** the width shrinks and the height is still capped by the shorter line, so the area can only decrease.

### Complexity

- **Time:** $O(n)$. **Space:** $O(1)$.

### Code

```python
from __future__ import annotations


class Solution:
    def maxArea(self, height: list[int]) -> int:
        lo, hi = 0, len(height) - 1
        best = 0
        while lo < hi:
            width = hi - lo
            if height[lo] < height[hi]:
                best = max(best, height[lo] * width)
                lo += 1          # lo is the bottleneck; no narrower partner can do better
            else:
                best = max(best, height[hi] * width)
                hi -= 1          # hi is the bottleneck (or tie: either side is safe to drop)
        return best


if __name__ == "__main__":
    s = Solution()
    assert s.maxArea([1, 8, 6, 2, 5, 4, 8, 3, 7]) == 49
    assert s.maxArea([1, 1]) == 1
    assert s.maxArea([4, 3, 2, 1, 4]) == 16
    assert s.maxArea([1, 2, 1]) == 2
    assert s.maxArea([0, 0]) == 0
    print("all tests passed")
```

**Self-grade:**

- ✅ Solved in 20 min **and** could explain why discarding the shorter side is safe → mastered.
- 🟡 Solved, but by intuition ("move the smaller one") without the proof → re-read Day 4 §1 and redo this problem on Day 14.
- 🔴 Not solved → redo LC 167 and LC 15 from scratch tomorrow before starting Week 2.

</details>

---

## E. Self-Audit & Gap Log (5 min)

Rate each item **1–5**: 1 = can't explain, 3 = can explain with notes, 5 = can teach it and spot it in code review.

| Area | Skill | Score |
|---|---|---|
| Memory | Explain refcounting vs cyclic GC, and when each frees an object | ☐ |
| Memory | Tell a leak from fragmentation using `tracemalloc` + RSS | ☐ |
| Memory | Spot `lru_cache`-on-method and stored-exception leaks in review | ☐ |
| Generators | Draw the 4-state machine; explain `send` / `throw` / `close` | ☐ |
| Generators | Explain `yield from` delegation and its return value | ☐ |
| Generators | Guarantee cleanup (`closing`, `aclosing`, external ownership) | ☐ |
| asyncio | Describe one loop iteration and how a Task drives a coroutine | ☐ |
| asyncio | Detect and fix loop blocking (debug mode, lag monitor, `to_thread`) | ☐ |
| asyncio | Safe background tasks; `TaskGroup` semantics | ☐ |
| DSA | Hashing: recognize membership / complement / frequency / grouping | ☐ |
| DSA | Two Pointers: explain the row/column elimination proof | ☐ |
| DSA | Sliding Window: state the monotonicity requirement and the "exactly K" trick | ☐ |

**Action rules:**

- Any **≤ 2** → redo that day's code example from a blank file on Day 8, before new material.
- Any **3** → add it to `GAPS.md` with a re-test date on Day 14.
- All **≥ 4** → stretch: solve LC 76 (Minimum Window Substring) this weekend.

**Retro (write 3 lines in `GAPS.md`):**

```text
Most surprising thing this week:
One production incident I now understand better:
One thing I'll do differently in Week 2:
```

---

[← Day 6 — Sliding Window](day-06-sliding-window.md) · [Week 1 overview](README.md) · [Week 2 — Day 8 →](../week-02/day-08-context-managers-exceptions.md)
