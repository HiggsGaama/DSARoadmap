# Day 14 — Consolidation & Self-Audit

[← Week 2](README.md) · **Track:** Audit · **Time box:** 60 min

| Block | Minutes | Purpose |
|---|---|---|
| A. Retrieval quiz (Week 2 + spaced Week 1) | 14 | Recall without notes |
| B. Pattern recognition drill | 8 | Map problem statements to patterns in under 30 s each |
| C. Production code review | 15 | Find this week's gotchas in realistic code |
| D. Transfer problem (timed) | 18 | A new problem with a known pattern, no hints |
| E. Self-audit & gap log | 5 | Decide what to revisit |

**Rules:** write your answer *before* opening each collapsible. Also re-test any Week 1 items in `GAPS.md` that were scheduled for today.

---

## A. Retrieval Quiz (14 min)

### Week 2

**Q1.** An `__exit__` method ends with `return self.committed`. What can go wrong, and why is it especially dangerous in async code?

<details markdown="1"><summary>Answer</summary>

Any truthy return value **suppresses** the in-flight exception, and that includes `BaseException` subclasses such as `asyncio.CancelledError` and `KeyboardInterrupt`. If `committed` is `True` when a cancellation arrives, the cancellation is swallowed. `asyncio.timeout`, `TaskGroup` sibling cancellation and graceful shutdown then stop working, and the task keeps running. `__exit__` should return `False` or `None` unless suppression is its documented purpose.

</details>

**Q2.** In a `@contextmanager` generator, where is the body's exception raised? What happens if the generator catches it, logs it, and doesn't re-raise?

<details markdown="1"><summary>Answer</summary>

The wrapper calls `gen.throw(exc)`, so the exception appears **at the `yield` line**. If the generator catches it and returns normally, `__exit__` returns `True` and the exception is **suppressed**. The caller continues as if the body had succeeded, often hitting an `UnboundLocalError` a few lines later. Cleanup must be in `finally`, and any `except` around `yield` must end in a bare `raise`.

</details>

**Q3.** Compare `raise X` inside `except`, `raise X from e`, and `raise X from None` in terms of `__context__`, `__cause__` and the printed traceback.

<details markdown="1"><summary>Answer</summary>

- Implicit: `__context__ = e`, `__cause__ = None`, printed as "During handling of the above exception, another exception occurred".
- `from e`: `__cause__ = e` and `__suppress_context__ = True`, printed as "The above exception was the direct cause…". Use it when translating errors.
- `from None`: `__cause__ = None` and `__suppress_context__ = True`. The original is hidden from the traceback but is still stored on `__context__`.

</details>

**Q4.** After refactoring `asyncio.gather` into `asyncio.TaskGroup`, an `except TimeoutError:` handler stopped firing. Why, and what's the fix?

<details markdown="1"><summary>Answer</summary>

`TaskGroup` wraps child failures in an `ExceptionGroup`, which isn't a `TimeoutError`. Use `except* TimeoutError as eg:` (with `eg.exceptions`), or inspect the group with `eg.subgroup(TimeoutError)`.

</details>

**Q5.** Recite the attribute lookup order for `obj.x`. Using it, explain why `property` can't be shadowed by the instance dict but `cached_property` can (and must be).

<details markdown="1"><summary>Answer</summary>

Data descriptor on the type → instance `__dict__` → non-data descriptor on the type → plain class attribute → `__getattr__`. `property` always defines `__set__`, which makes it a **data** descriptor that wins over the dict. `cached_property` defines only `__get__` (non-data). It stores the computed value in `obj.__dict__` under the same name, and from then on step 2 finds the value directly. That's what makes later accesses free, and it's also why the value goes stale.

</details>

**Q6.** In `class EmbeddingClient(Retrying, Metered)`, where both classes inherit from `Service`, what does `super().__init__(**kwargs)` inside `Retrying.__init__` call? What breaks if `Retrying` calls `Service.__init__(self)` instead?

<details markdown="1"><summary>Answer</summary>

It calls `Metered.__init__`, the next class after `Retrying` in `EmbeddingClient.__mro__` (`EmbeddingClient → Retrying → Metered → Service → object`). Calling `Service.__init__` directly skips `Metered` entirely, so its setup never runs and its keyword arguments end up being rejected or lost.

</details>

**Q7.** When do Python threads give real parallel speed-up on a standard (GIL) build?

<details markdown="1"><summary>Answer</summary>

When threads spend their time with the GIL **released**: in blocking I/O system calls, or in C/Rust extensions doing long native work (NumPy/BLAS, hashlib on large buffers, compression, PyTorch ops, tokenizers). Pure-Python CPU loops get no speed-up and are often slower because of hand-off overhead.

</details>

**Q8.** Why can `multiprocessing.Pool()` hang in a service that has a background metrics thread on Linux ≤ 3.13? Give two fixes.

<details markdown="1"><summary>Answer</summary>

The default start method is `fork`, which copies every lock's state but only the calling thread. A lock held by the metrics thread at fork time (for example the logging handler lock or the malloc arena lock) stays locked forever in the child, which deadlocks on first use. Fixes: use `get_context("forkserver")` or `"spawn"` explicitly; create pools before starting any threads; start background threads in a post-fork hook.

</details>

**Q9.** In the template `while lo < hi: mid = (lo + hi) // 2; if pred(mid): hi = mid else: lo = mid + 1`, why does the loop always terminate? What are Koko's bounds, and why?

<details markdown="1"><summary>Answer</summary>

`mid` is the lower middle, so `lo <= mid < hi`. Both branches strictly shrink `[lo, hi]`, either by moving `hi` down to `mid` or by moving `lo` past `mid`. Koko's range is `[1, max(piles)]`. Speed 0 is meaningless (and divides by zero). At `max(piles)` every pile takes exactly 1 hour, and `h >= n` guarantees that's feasible, so going faster can't help.

</details>

**Q10.** Sketch why Floyd's phase 2 meets at the cycle entrance.

<details markdown="1"><summary>Answer</summary>

At the meeting point, slow has walked $a + b$ and fast has walked $2(a + b) = a + b + kL$, so $a = (k - 1)L + (L - b)$. A pointer starting from the head walks $a$ steps to the entrance. A pointer starting at the meeting point walks the same $a$ steps, which is $k - 1$ full laps plus the $L - b$ steps remaining to the entrance. They arrive together.

</details>

### Spaced repetition: Week 1

**Q11.** Why does `@lru_cache` on a method leak, and what's the preferred fix?

<details markdown="1"><summary>Answer</summary>

The cache belongs to the class-level function and its keys include `self`, so it holds strong references to every instance (and everything they reference) until eviction, which never happens with `maxsize=None`. Prefer a module-level pure function keyed on primitives, or a per-instance cache that dies with the instance.

</details>

**Q12.** A consumer `break`s out of `for row in rows(conn):`. When does the generator's `finally: cursor.close()` run, and how do you make it deterministic?

<details markdown="1"><summary>Answer</summary>

Only when the generator is closed: when the last reference disappears (refcounting), or later if a reference survives, the generator is in a cycle, or it's an async generator. Make it deterministic with `with contextlib.closing(gen):` (or `aclosing` for async generators), or by keeping resource ownership outside the generator.

</details>

**Q13.** Why must you keep a reference to the return value of `asyncio.create_task()` for fire-and-forget work?

<details markdown="1"><summary>Answer</summary>

The event loop keeps only weak references to tasks. Without a strong reference, a suspended task can be garbage-collected before it finishes, and its exception is never retrieved. Keep a set plus `add_done_callback(set.discard)`, or use a `TaskGroup`.

</details>

---

## B. Pattern Recognition Drill (8 min)

For each statement, name the pattern and the **one phrase** that gave it away.

1. "For each building, find the index of the nearest taller building to its left."
2. "Find the minimum ship capacity that ships all packages, in order, within D days."
3. "Return the middle node of a singly linked list in one pass."
4. "An array of n+1 integers, each in `[1, n]`. Find the duplicate without modifying the array, using O(1) extra space."
5. "Return the maximum value in every contiguous window of size k."
6. "Find the area of the largest rectangle in a histogram."
7. "Remove the k-th node from the end of a linked list in one pass."
8. "Given a sorted array of distinct integers, return the index where `target` is, or where it would be inserted."

<details markdown="1"><summary>Answers</summary>

| # | Pattern | Giveaway |
|---|---|---|
| 1 | Monotonic stack (previous greater): after popping, the remaining top is the answer | "nearest taller", "to its left" |
| 2 | Binary search on the answer, range `[max(w), sum(w)]`, greedy day count as the predicate | "minimum capacity", "within D days" |
| 3 | Fast/slow pointers | "middle", "one pass" |
| 4 | Floyd cycle detection on the function graph `i -> nums[i]` | "O(1) space", "don't modify", values in `[1, n]` |
| 5 | Monotonic **deque** (sliding window maximum), not a plain stack | "every window of size k", "maximum" |
| 6 | Monotonic stack (increasing): a pop fixes that bar's width | "largest rectangle", "histogram" |
| 7 | Gap two pointers: lead by k, then move together | "k-th from the end", "one pass" |
| 8 | Binary search, lower bound (`bisect_left`) | "sorted", "would be inserted" |

</details>

---

## C. Production Code Review (15 min)

This ingestion job embeds batches of rows on GPUs. In production, `processed` is always logged as 0 and the job returns 0. Some batches silently produce no vectors and an `UnboundLocalError` shows up in the logs. Retry limits act strangely across batches. The size check approves oversized batches, and the job occasionally hangs forever on Linux. **Find at least seven defects**, then write the fix before opening the answer.

```python
import logging
import multiprocessing as mp
import threading
from contextlib import contextmanager
from functools import cached_property

log = logging.getLogger("ingest")


class NonNegative:
    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        return self.value

    def __set__(self, obj, value):
        if value < 0:
            raise ValueError(f"{self.name} must be >= 0")
        self.value = value


class Batch:
    max_retries = NonNegative()

    def __init__(self, rows: list[dict], max_retries: int) -> None:
        self.rows = rows
        self.max_retries = max_retries
        log.debug("new batch, %d bytes", self.size_bytes)

    @cached_property
    def size_bytes(self) -> int:
        return sum(len(str(r)) for r in self.rows)


@contextmanager
def gpu_lease(pool):
    device = pool.acquire()
    try:
        yield device
    except Exception:
        log.exception("batch failed on %s", device)
    pool.release(device)


processed = 0


def embed_batch(batch: Batch) -> int:
    global processed
    with gpu_lease(GPU_POOL) as device:
        vectors = device.embed(batch.rows)
    processed += len(vectors)
    return len(vectors)


def run(batches: list[Batch]) -> int:
    threading.Thread(target=push_metrics_forever, daemon=True).start()
    for b in batches:
        b.rows.extend(load_late_rows(b))           # rows added after construction
        if b.size_bytes > 50_000_000:
            raise ValueError("batch too large")
    with mp.Pool(4) as pool:
        counts = pool.map(embed_batch, batches)
    try:
        return sum(counts)
    finally:
        log.info("processed=%d", processed)
        return processed
```

<details markdown="1"><summary>Defects & fixed version</summary>

| # | Defect | Consequence | Day |
|---|---|---|---|
| 1 | `NonNegative` stores `self.value` on the descriptor | All `Batch` instances share **one** `max_retries`, so the last one constructed wins | 10 |
| 2 | `cached_property` computed in `__init__`, rows extended later | `size_bytes` is stale; oversized batches pass the check | 10 |
| 3 | `except Exception` around `yield` without `raise` | Errors are **suppressed**, so `vectors` is never assigned → `UnboundLocalError` hides the real failure | 8 |
| 4 | `pool.release` not in `finally` | Leaked on `BaseException` (`KeyboardInterrupt`, worker termination) | 8 |
| 5 | Module-global `processed` incremented in worker **processes** | Each child has its own copy; the parent's stays 0. (With threads it would be a race instead) | 12 |
| 6 | Background thread started, then `mp.Pool` with the Linux ≤ 3.13 default `fork` | Children can inherit a lock held by the metrics or logging thread → **intermittent hang** | 12 |
| 7 | `return` inside `finally` | Overrides `sum(counts)` with 0 **and** would swallow any exception from the `try` | 8 |
| 8 | GPU device handles used in forked children (bonus) | CUDA can't be re-initialized in a forked subprocess (`spawn` is required for CUDA in worker processes) | 12 |
| 9 | Each `Batch` (rows plus the cached value) is pickled per task (bonus) | High IPC cost; pass IDs or paths and load inside the worker | 12 |

```python
from __future__ import annotations

import logging
import multiprocessing as mp
from collections.abc import Iterator
from contextlib import contextmanager
from typing import Any

log = logging.getLogger("ingest")


class NonNegative:
    def __set_name__(self, owner: type, name: str) -> None:
        self.name = name

    def __get__(self, obj: object | None, objtype: type | None = None) -> Any:
        if obj is None:
            return self
        return vars(obj)[self.name]                  # per-instance storage

    def __set__(self, obj: object, value: int) -> None:
        if value < 0:
            raise ValueError(f"{self.name} must be >= 0")
        vars(obj)[self.name] = value


class Batch:
    max_retries = NonNegative()

    def __init__(self, batch_id: str, rows: list[dict[str, Any]], max_retries: int) -> None:
        self.batch_id = batch_id
        self.rows = rows
        self.max_retries = max_retries

    @property                                        # cheap enough to recompute; never stale
    def size_bytes(self) -> int:
        return sum(len(str(r)) for r in self.rows)


@contextmanager
def gpu_lease(pool: Any) -> Iterator[Any]:
    device = pool.acquire()
    try:
        yield device
    except Exception:
        log.exception("batch failed on %s", device)
        raise                                        # annotate, never swallow
    finally:
        pool.release(device)


def embed_batch(batch_id: str) -> int:              # receives an ID, loads data in the worker
    batch = load_batch(batch_id)
    with gpu_lease(worker_gpu_pool()) as device:
        vectors = device.embed(batch.rows)
    return len(vectors)                              # results flow back via return values


def run(batch_ids: list[str]) -> int:
    ctx = mp.get_context("spawn")                    # CUDA-safe; immune to inherited locks
    with ctx.Pool(4) as pool:
        counts = pool.map(embed_batch, batch_ids)
    processed = sum(counts)
    log.info("processed=%d", processed)
    return processed

# Size validation moves to where rows are final (inside load_batch or right before submit).
# Start push_metrics_forever only in the parent AFTER pools exist, or use spawn/forkserver (done).
```

</details>

---

## D. Transfer Problem — LeetCode 153: Find Minimum in Rotated Sorted Array (Medium) · 18 min timed

No hints this time. Identify the pattern yourself.

**Statement:** An ascending array of **unique** integers was rotated between 1 and `n` times (for example, `[0,1,2,4,5,6,7]` became `[4,5,6,7,0,1,2]`). Given the rotated array `nums`, return its minimum element. **You must write an algorithm that runs in $O(\log n)$ time.**

```text
Input:  nums = [3, 4, 5, 1, 2]
Output: 1

Input:  nums = [4, 5, 6, 7, 0, 1, 2]
Output: 0

Input:  nums = [11, 13, 15, 17]
Output: 11          # rotated n times = not rotated
```

**Constraints:** `1 <= n <= 5000`, `-5000 <= nums[i] <= 5000`, all values unique.

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Pattern

**Binary search on a monotone predicate** (Day 11), on an array that *isn't* globally sorted. That's the transfer: find the predicate instead of looking for sorted data.

### Intuition

1. The rotated array is two ascending runs: a "high" run followed by a "low" run. Every element of the low run is `<= nums[-1]`. Every element of the high run is `> nums[-1]`.
2. So `pred(i) = nums[i] <= nums[-1]` is `False … False True … True`, and the **first True** is the minimum.
3. Equivalently, compare `nums[mid]` with `nums[hi]`. If `nums[mid] > nums[hi]`, the drop is strictly to the right of `mid`, so set `lo = mid + 1`. Otherwise `mid` is in the low run and might be the minimum, so set `hi = mid`.
4. Comparing with `nums[lo]` doesn't work cleanly: in an unrotated array `nums[mid] > nums[lo]` holds and yet the minimum is on the left. Compare with the right end.

### Complexity

- **Time:** $O(\log n)$. **Space:** $O(1)$.

### Code

```python
from __future__ import annotations


class Solution:
    def findMin(self, nums: list[int]) -> int:
        lo, hi = 0, len(nums) - 1
        while lo < hi:
            mid = (lo + hi) // 2
            if nums[mid] > nums[hi]:
                lo = mid + 1      # mid is in the high run; the drop is to its right
            else:
                hi = mid          # mid is in the low run; it might be the minimum
        return nums[lo]


if __name__ == "__main__":
    s = Solution()
    assert s.findMin([3, 4, 5, 1, 2]) == 1
    assert s.findMin([4, 5, 6, 7, 0, 1, 2]) == 0
    assert s.findMin([11, 13, 15, 17]) == 11
    assert s.findMin([1]) == 1
    assert s.findMin([2, 1]) == 1
    base = list(range(-50, 50))
    for k in range(len(base)):                       # every rotation
        assert s.findMin(base[k:] + base[:k]) == -50
    print("all tests passed")
```

**Self-grade:**

- ✅ Solved in 18 min **and** could state the predicate and why you compare with `nums[hi]` → mastered.
- 🟡 Solved by pattern-matching from memory without the predicate → redo Day 11 §1 and solve LC 33 (Search in Rotated Sorted Array) on Day 21.
- 🔴 Not solved → re-derive the template on paper, then redo LC 875 and this problem tomorrow.

</details>

---

## E. Self-Audit & Gap Log (5 min)

Rate each item **1–5**: 1 = can't explain, 3 = can explain with notes, 5 = can teach it and spot it in code review.

| Area | Skill | Score |
|---|---|---|
| Context managers | Write out what `with` desugars to; state the suppression rule | ☐ |
| Context managers | Write a leak-proof `@contextmanager`; use `ExitStack.pop_all()` | ☐ |
| Exceptions | `__cause__` vs `__context__`; `except*` with `TaskGroup`; `return`-in-`finally` | ☐ |
| Descriptors | Recite the lookup order; data vs non-data | ☐ |
| Descriptors | Per-instance storage via `__set_name__`; `cached_property` semantics | ☐ |
| MRO | Predict `super()` targets in a diamond; cooperative `**kwargs` | ☐ |
| Concurrency | Choose asyncio / threads / processes for a given workload | ☐ |
| Concurrency | Explain the fork-after-threads deadlock; pick start methods deliberately | ☐ |
| Concurrency | Avoid per-task pickling; worker initializers | ☐ |
| DSA | Monotonic stack: pick the stack order and pop condition for any "next/previous greater/smaller" | ☐ |
| DSA | Binary search: "first True" template, answer-space bounds | ☐ |
| DSA | Floyd: both phases and the proof | ☐ |

**Action rules:**

- Any **≤ 2** → redo that day's code example from a blank file on Day 15, before new material.
- Any **3** → add it to `GAPS.md` with a re-test date on Day 21.
- All **≥ 4** → stretch: LC 84 (Largest Rectangle in Histogram) and LC 410 (Split Array Largest Sum).

**Retro (write 3 lines in `GAPS.md`):**

```text
Most surprising thing this week:
One production incident I now understand better:
One thing I'll do differently in Week 3:
```

---

[← Day 13 — Fast/Slow Pointers](day-13-linked-lists-fast-slow.md) · [Week 2 overview](README.md) · [Day 15 — Metaclasses →](../week-03/day-15-metaclasses-class-creation.md)
