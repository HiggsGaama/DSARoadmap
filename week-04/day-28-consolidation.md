# Day 28 — Consolidation & Self-Audit

[← Week 4](README.md) · **Track:** Audit · **Time box:** 60 min

| Block | Minutes | Purpose |
|---|---|---|
| A. Retrieval quiz (+ spaced repetition) | 14 | Recall Week 4 without notes; re-test Weeks 1–3 |
| B. Pattern recognition drill | 6 | Map problem statements to patterns in under 30 s each |
| C. Production code review | 15 | Find this week's gotchas in a realistic inference service |
| D. Transfer problem (timed) | 20 | A new problem with a known pattern, no hints |
| E. Self-audit & gap log | 5 | Decide what to revisit |

**Rules:** write your answer *before* opening each collapsible. A wrong answer you wrote down teaches more than a right answer you only recognized.

---

## A. Retrieval Quiz (14 min)

**Q1.** You call `task.cancel()`. When, and where, does the task actually see `CancelledError`? Why can't an `asyncio.timeout()` interrupt a 2-second sync loop?

<details markdown="1"><summary>Answer</summary>

`cancel()` only sets a flag, increments `cancelling()`, and cancels the future the task is parked on. On the task's next step the loop calls `coro.throw(CancelledError())`, so it's raised at the `await` where the coroutine is suspended. A timeout is itself just a timer callback that calls `task.cancel()`. The loop is single-threaded, so neither the timer nor the delivery can run until the sync code returns control at an `await`.

</details>

**Q2.** `CancelledError` became a `BaseException` in 3.8. Name three ways code still swallows it, and what goes wrong in production when it does.

<details markdown="1"><summary>Answer</summary>

Bare `except:`, `except BaseException:`, `except asyncio.CancelledError:` without re-raise, `contextlib.suppress(BaseException)`, or `return`/`break` inside `finally`. Consequences: `asyncio.timeout` never raises `TimeoutError` (the caller thinks the call succeeded), shutdown hangs until `SIGKILL`, and `TaskGroup`s can't unwind.

</details>

**Q3.** How does `asyncio.timeout()` tell *its own* cancellation apart from an external one, for example a server shutdown cancelling a request that's inside a timeout block?

<details markdown="1"><summary>Answer</summary>

It records `task.cancelling()` on entry. On exit, if its timer fired and the exception is `CancelledError`, it calls `task.uncancel()`. If the counter drops back to the entry value, the cancellation was its own, so it converts it into `TimeoutError`. If another cancellation is still pending (the counter stays higher), it lets `CancelledError` propagate.

</details>

**Q4.** `TaskGroup` vs `asyncio.gather()`: one child raises `ValueError`. What happens to the siblings, and what does the caller receive?

<details markdown="1"><summary>Answer</summary>

**TaskGroup:** cancels all siblings and the body, waits for their cleanup, then raises an `ExceptionGroup` containing the `ValueError` (handle it with `except* ValueError`). **gather (default):** the `ValueError` propagates to the caller immediately, and the siblings **keep running** unsupervised. With `return_exceptions=True`, exceptions are returned as values, and still nothing gets cancelled.

</details>

**Q5.** Your model sustains 200 req/s and the latency budget allows 0.5 s of queueing. Roughly how deep should the request queue be, and what should happen to request number (bound + 1)?

<details markdown="1"><summary>Answer</summary>

By Little's law, L = λ × W = 200 × 0.5 ≈ **100**. Anything deeper is guaranteed to miss the SLO. Request bound + 1 should be **rejected immediately** (`put_nowait` → `QueueFull` → `503` + `Retry-After`), not queued, so the client can retry elsewhere and the server doesn't burn GPU time on answers nobody will wait for.

</details>

**Q6.** In a dynamic batcher, what does `max_wait_ms` cost at low traffic and at high traffic? What two checks must the batcher make on each request's future?

<details markdown="1"><summary>Answer</summary>

At low traffic, every request can wait up to the full `max_wait` window. At high traffic, batches fill to `max_batch_size` instantly, so it costs nothing. The batcher must check `fut.done()` **before inference** (skip callers who timed out or disconnected, to avoid wasted GPU work) and **before `set_result`/`set_exception`** (a future cancelled mid-inference raises `InvalidStateError`).

</details>

**Q7.** In a `cProfile` report, a function has a large `tottime` but calls nothing expensive. What's the likely explanation? And why is `py-spy` the right tool for a slow production pod?

<details markdown="1"><summary>Answer</summary>

cProfile is function-granular, so inline bytecode work (`x in big_list`, arithmetic, attribute access in a loop) is charged to the enclosing function's own time. py-spy samples from outside the process with low, uniform overhead and no code change or restart. Use `py-spy dump` for a hung worker and `record` for a flame graph.

</details>

**Q8.** Why report the **minimum** of `timeit` repeats, and why never measure durations with `time.time()`?

<details markdown="1"><summary>Answer</summary>

Noise (scheduling, caches, frequency scaling) only ever *adds* time, so the minimum best estimates the intrinsic cost. Make sure each repeat measures the same thing, for example by clearing caches in `setup`. `time.time()` is wall-clock time: it can jump with NTP adjustments and may have coarse resolution (about 16 ms on some Windows builds). `perf_counter()` is monotonic and high-resolution.

</details>

**Q9.** For the **k largest** items in a stream, which heap do you keep, of what size, and why? What breaks with `heappush(h, (score, request_dict))`?

<details markdown="1"><summary>Answer</summary>

A **min-heap of size k**. Its root is the smallest of the current top k, so a newcomer only gets in if it beats the root (`heapreplace`). That's $O(n \log k)$ time and $O(k)$ memory. When two scores tie, tuple comparison falls through to the dicts and raises `TypeError`. Add a tiebreaker: `(score, next(counter), payload)`.

</details>

**Q10.** Merge Intervals: why `max(block_end, cur_end)` and not `cur_end`? In a sweep line over half-open meetings, why must an end at time t be processed before a start at t?

<details markdown="1"><summary>Answer</summary>

A fully contained interval (`[1,10]` then `[2,3]`) would otherwise shrink the block. With half-open `[start, end)`, a meeting ending at 10:00 frees the room for one starting at 10:00. Processing `−1` first (sorting `(time, delta)` does this, since `−1 < +1`) avoids overcounting concurrency.

</details>

**Q11.** Grid BFS: why mark cells visited when **enqueuing** rather than when dequeuing? Why does recursive DFS fail on a 300 × 300 all-land grid?

<details markdown="1"><summary>Answer</summary>

Marking on dequeue lets several neighbours push the same cell before it's popped, so the queue fills with duplicates and cells get processed more than once. Recursive DFS on a single 90,000-cell component exceeds Python's default recursion limit (~1000) → `RecursionError`. Raising the limit risks a C-stack crash. Use a `deque` BFS or an explicit stack.

</details>

### Spaced repetition: Weeks 1–3

**SR1 (Week 2, descriptors).** A class defines `total` as a `@property` and `summary` as a `functools.cached_property`. Someone runs `obj.__dict__["total"] = 0; obj.__dict__["summary"] = "x"`. What do `obj.total` and `obj.summary` return, and why?

<details markdown="1"><summary>Answer</summary>

`obj.total` still calls the property, because `property` defines `__set__`, which makes it a **data descriptor**, and data descriptors take precedence over the instance `__dict__`. `obj.summary` returns `"x"`: `cached_property` is a **non-data descriptor** (only `__get__`), so the instance `__dict__` wins. That's exactly how `cached_property` caches, and also how it goes stale or gets shadowed. The lookup order is data descriptor → instance dict → non-data descriptor → class attribute → `__getattr__`.

</details>

**SR2 (Week 2, GIL / Week 4, profiling).** An async service tokenizes 5 MB documents in pure Python, which takes about 300 ms of CPU each. A colleague wraps the call in `asyncio.to_thread`. Does it fix p99? What's the better fix, and what would py-spy show before the change?

<details markdown="1"><summary>Answer</summary>

`to_thread` stops the *event loop* from blocking outright, but pure-Python CPU work holds the GIL, so the loop thread still gets only part of the CPU time (the switch interval is 5 ms), and the tokenizations don't run in parallel. Latency still degrades under load. The better fix is a `ProcessPoolExecutor` (or a separate worker service, or a tokenizer implemented in Rust/C that releases the GIL; on free-threaded 3.13t+ builds, threads can run truly in parallel). Before the change, `py-spy dump` would show the event-loop thread inside the tokenizer frames rather than in `select`.

</details>

**SR3 (DSA, Weeks 2–3).** Which pattern does each statement call for? (a) "For each day, how many days until a warmer temperature?" (b) "Minimum ship capacity so that all packages ship within D days." (c) "Is this binary tree a valid BST?"

<details markdown="1"><summary>Answer</summary>

(a) **Monotonic stack** (decreasing stack of indices: "next greater element"). (b) **Binary search on the answer**: the predicate "can ship within D days with capacity C" is monotone in C, so find the first True. (c) **Tree DFS carrying `(low, high)` bounds** (or an in-order traversal that must be strictly increasing). Checking only parent and child is the classic bug.

</details>

---

## B. Pattern Recognition Drill (6 min)

Name the pattern and the giveaway phrase for each.

1. "Return the k points closest to the origin."
2. "Given meeting start/end times, return the minimum number of rooms required."
3. "Each minute, fire spreads to adjacent dry cells from several initial fires. When is everything burnt?"
4. "Count connected regions of same-coloured pixels in an image."
5. "Remove the minimum number of intervals so that the rest don't overlap."
6. "Report the median latency after each new measurement arrives."
7. "Merge k sorted log files into one stream ordered by timestamp."
8. "Shortest path from the top-left to the bottom-right of a 0/1 matrix, moving in 8 directions."

<details markdown="1"><summary>Answers</summary>

| # | Pattern | Giveaway |
|---|---|---|
| 1 | Heap of size k: **max-heap** (negated distance), evict the farthest | "k closest" |
| 2 | Sweep line, or a min-heap of end times | "minimum number of rooms" = peak concurrency |
| 3 | **Multi-source BFS**, layer = minute | "several initial", "each minute", "when" |
| 4 | Grid DFS/BFS connected components (flood fill) | "connected regions" |
| 5 | Sort by **end** + greedy keep | "minimum removals", "non-overlapping" |
| 6 | Two heaps (lower max-heap, upper min-heap) | "median", "after each new" (streaming) |
| 7 | Min-heap of the current head of each file (`heapq.merge`) | "k sorted", "merge" |
| 8 | BFS (unweighted shortest path) | "shortest path", grid, uniform step cost |

</details>

---

## C. Production Code Review (15 min)

This batching inference service passes its unit tests. In production: during a traffic spike it gets OOM-killed; on deploys pods hang until `SIGKILL`; after a single bad request a handful of clients hang forever; and the "top slow requests" endpoint occasionally returns `500 TypeError`. **Find at least six defects.**

```python
import asyncio
import heapq
import time

queue: asyncio.Queue = asyncio.Queue()


async def infer_batch(prompts: list[str]) -> list[str]: ...


async def batcher() -> None:
    while True:
        try:
            batch = [await queue.get()]
            while not queue.empty() and len(batch) < 32:
                batch.append(queue.get_nowait())
            start = time.time()
            results = await infer_batch([prompt for prompt, _ in batch])
            for (_, fut), result in zip(batch, results):
                fut.set_result(result)
            print("batch took", time.time() - start)
        except:
            continue


async def predict(prompt: str) -> str:
    fut = asyncio.get_running_loop().create_future()
    await queue.put((prompt, fut))
    return await fut


def slowest_requests(stats: list[dict], k: int) -> list[dict]:
    heap: list = []
    for s in stats:
        heapq.heappush(heap, (s["latency_ms"], s))
        if len(heap) > k:
            heapq.heappop(heap)
    return [s for _, s in heap]


async def shutdown(batcher_task: asyncio.Task) -> None:
    batcher_task.cancel()
    await batcher_task
```

<details markdown="1"><summary>Defects & fixed version</summary>

| # | Defect | Consequence | Day |
|---|---|---|---|
| 1 | `asyncio.Queue()` is unbounded, and `predict` uses a blocking `put` | Spikes pile up in memory → latency climbs → OOM. No load shedding | 24 |
| 2 | Bare `except:` swallows `CancelledError` | `batcher_task.cancel()` is ignored and `shutdown` awaits forever, so deploys hang until `SIGKILL` | 22 |
| 3 | On an inference exception, the batch's futures are never resolved | Every caller in that batch hangs forever (there's no timeout either) | 24 |
| 4 | `fut.set_result` on a future whose caller was cancelled | `InvalidStateError`, which the bare `except` swallows, so the **rest of the batch** never gets results | 22/24 |
| 5 | `await fut` with no deadline | A stuck model call or a lost future pins the request forever | 22 |
| 6 | `(latency, dict)` heap entries | Equal latencies → dict comparison → `TypeError` → the 500s | 23 |
| 7 | `time.time()` for durations | Coarse resolution, can jump with NTP. Use `perf_counter()` | 26 |
| 8 | `slowest_requests` returns heap order | The output isn't sorted, which surprises callers. Sort descending at the end | 23 |
| 9 | `zip` without `strict=True` | A model returning fewer results silently leaves futures unresolved | bonus |

Fixed version (runnable):

```python
from __future__ import annotations

import asyncio
import heapq
import itertools
import time
from collections.abc import Sequence
from dataclasses import dataclass
from typing import Any


class Overloaded(Exception):
    """Map to HTTP 503 + Retry-After."""


@dataclass(slots=True)
class Pending:
    prompt: str
    future: asyncio.Future[str]


async def infer_batch(prompts: Sequence[str]) -> list[str]:
    await asyncio.sleep(0.01)
    if "boom" in prompts:
        raise RuntimeError("model crashed on this batch")
    return [p.upper() for p in prompts]


async def batcher(queue: asyncio.Queue[Pending], max_batch: int = 32) -> None:
    live: list[Pending] = []
    try:
        while True:
            batch = [await queue.get()]
            while not queue.empty() and len(batch) < max_batch:
                batch.append(queue.get_nowait())
            live = [p for p in batch if not p.future.done()]       # skip abandoned callers
            if not live:
                continue
            start = time.perf_counter()
            try:
                results = await infer_batch([p.prompt for p in live])
            except Exception as exc:                               # NOT bare: cancel propagates
                for p in live:
                    if not p.future.done():
                        p.future.set_exception(exc)
                continue
            for p, result in zip(live, results, strict=True):
                if not p.future.done():
                    p.future.set_result(result)
            print(f"  batch of {len(live)} took {(time.perf_counter() - start) * 1000:.0f} ms")
    finally:
        while not queue.empty():                                   # fail everything pending
            live.append(queue.get_nowait())
        for p in live:
            if not p.future.done():
                p.future.set_exception(Overloaded("shutting down"))


async def predict(queue: asyncio.Queue[Pending], prompt: str, timeout_s: float = 2.0) -> str:
    fut: asyncio.Future[str] = asyncio.get_running_loop().create_future()
    try:
        queue.put_nowait(Pending(prompt, fut))
    except asyncio.QueueFull:
        raise Overloaded("queue full") from None
    async with asyncio.timeout(timeout_s):
        return await fut


def slowest_requests(stats: list[dict[str, Any]], k: int) -> list[dict[str, Any]]:
    tiebreak = itertools.count()
    heap: list[tuple[float, int, dict[str, Any]]] = []
    for s in stats:
        entry = (s["latency_ms"], next(tiebreak), s)
        if len(heap) < k:
            heapq.heappush(heap, entry)
        elif entry[0] > heap[0][0]:
            heapq.heapreplace(heap, entry)
    return [s for _, _, s in sorted(heap, reverse=True)]


async def main() -> None:
    queue: asyncio.Queue[Pending] = asyncio.Queue(maxsize=64)
    task = asyncio.create_task(batcher(queue))
    results = await asyncio.gather(
        *(predict(queue, p) for p in ["a", "b", "boom", "c"]), return_exceptions=True
    )
    print("  results:", [r if isinstance(r, str) else type(r).__name__ for r in results])
    results = await asyncio.gather(*(predict(queue, p) for p in ["d", "e"]))
    print("  next batch unaffected:", results)

    stats = [{"id": i, "latency_ms": ms} for i, ms in enumerate([50, 90, 90, 10, 90, 70])]
    print("  slowest 3:", [s["id"] for s in slowest_requests(stats, 3)])

    task.cancel()                                                  # shutdown completes now
    await asyncio.gather(task, return_exceptions=True)
    print("  batcher cancelled cleanly:", task.cancelled())


if __name__ == "__main__":
    asyncio.run(main())
```

Expected output:

```text
  results: ['RuntimeError', 'RuntimeError', 'RuntimeError', 'RuntimeError']
  batch of 2 took 16 ms
  next batch unaffected: ['D', 'E']
  slowest 3: [4, 2, 1]
  batcher cancelled cleanly: True
```

The `"boom"` prompt fails its *whole batch* (all four callers get the `RuntimeError` promptly instead of hanging), and the batcher keeps serving the next batch. The three requests tied at 90 ms no longer crash the heap. Timings vary.

</details>

---

## D. Transfer Problem — LeetCode 994: Rotting Oranges (Medium) · 20 min timed

No hints. Identify the pattern yourself.

**Statement:** In an `m × n` grid, each cell is `0` (empty), `1` (fresh orange) or `2` (rotten orange). Every minute, any fresh orange **4-directionally adjacent** to a rotten orange becomes rotten. Return the minimum number of minutes until no fresh orange remains. If that's impossible, return `-1`.

```text
Input:  grid = [[2,1,1],[1,1,0],[0,1,1]]
Output: 4

Input:  grid = [[2,1,1],[0,1,1],[1,0,1]]
Output: -1          # bottom-left orange can never be reached

Input:  grid = [[0,2]]
Output: 0           # no fresh oranges at all
```

**Constraints:** `1 <= m, n <= 10`.

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Pattern

**Multi-source BFS.** "Every minute, spreads to adjacent cells" means BFS layers are minutes. "Several initially rotten oranges" means you seed the queue with **all** of them at minute 0. Running a separate BFS from each rotten orange and taking minimums would be both slower and more complicated.

### Intuition

1. Scan once: enqueue every rotten cell and count the fresh ones.
2. Process the BFS **level by level** (`for _ in range(len(queue))`). Each level is one minute. Rot fresh neighbours as you enqueue them (mark on enqueue, Day 27) and decrement `fresh`.
3. Only count a minute if that level actually rotted something. Otherwise you'd report one extra minute for the final, unproductive level.
4. Return `minutes` if `fresh == 0`, else `-1`. Remaining fresh oranges are unreachable.

### Complexity

- **Time:** $O(R \cdot C)$. Each cell is enqueued at most once. **Space:** $O(R \cdot C)$ for the queue in the worst case.

### Code

```python
from __future__ import annotations

from collections import deque


class Solution:
    def orangesRotting(self, grid: list[list[int]]) -> int:
        rows, cols = len(grid), len(grid[0])
        state = [row[:] for row in grid]          # don't mutate the caller's grid
        queue: deque[tuple[int, int]] = deque()
        fresh = 0
        for r in range(rows):
            for c in range(cols):
                if state[r][c] == 2:
                    queue.append((r, c))          # ALL sources at minute 0
                elif state[r][c] == 1:
                    fresh += 1

        minutes = 0
        while queue and fresh:
            for _ in range(len(queue)):           # exactly one BFS layer = one minute
                r, c = queue.popleft()
                for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                    nr, nc = r + dr, c + dc
                    if 0 <= nr < rows and 0 <= nc < cols and state[nr][nc] == 1:
                        state[nr][nc] = 2         # mark on enqueue
                        fresh -= 1
                        queue.append((nr, nc))
            minutes += 1                          # loop guard ensures this layer was productive

        return minutes if fresh == 0 else -1


if __name__ == "__main__":
    s = Solution()
    assert s.orangesRotting([[2, 1, 1], [1, 1, 0], [0, 1, 1]]) == 4
    assert s.orangesRotting([[2, 1, 1], [0, 1, 1], [1, 0, 1]]) == -1
    assert s.orangesRotting([[0, 2]]) == 0
    assert s.orangesRotting([[0]]) == 0
    assert s.orangesRotting([[1]]) == -1                      # fresh, no rotten source
    assert s.orangesRotting([[2, 1, 1, 1, 2]]) == 2           # two sources meet in the middle
    print("all tests passed")
```

**Why `while queue and fresh`:** once `fresh` reaches 0 you stop, so the last increment always corresponds to a minute in which something rotted. That handles the off-by-one in step 3 without a separate flag.

**Self-grade:**

- ✅ Solved in 20 min **with** multi-source seeding and level counting → mastered.
- 🟡 Solved, but with a BFS per rotten orange, or an off-by-one in minutes → re-read Day 27 and redo LC 542 (01 Matrix) on Day 35.
- 🔴 Not solved → redo LC 200 from scratch (BFS version) before Day 29.

</details>

---

## E. Self-Audit & Gap Log (5 min)

Rate each item **1–5**: 1 = can't explain, 3 = can explain with notes, 5 = can teach it and spot it in code review.

| Area | Skill | Score |
|---|---|---|
| Cancellation | Explain delivery at `await`, `cancelling()`/`uncancel()`, and how `timeout()` converts errors | ☐ |
| Cancellation | Spot swallowed `CancelledError`; use `shield` correctly; know `to_thread` threads keep running | ☐ |
| Structured concurrency | `TaskGroup` vs `gather` semantics; a graceful SIGTERM shutdown sequence | ☐ |
| Backpressure | Size a queue bound with Little's law; block inside the pipeline vs shed at the edge | ☐ |
| Micro-batching | Implement `max_batch` + `max_wait` with per-request futures; handle cancelled futures and shutdown | ☐ |
| Profiling | Choose cProfile vs py-spy vs memray; read `tottime`/`cumtime` and flame graphs | ☐ |
| Benchmarking | `perf_counter`, min-of-repeats, cold vs warm caches, Amdahl's law | ☐ |
| DSA | Heaps: size-k min-heap for top-k; tuple tie-breakers; quickselect trade-offs | ☐ |
| DSA | Intervals: sort-by-start merge, sort-by-end greedy, sweep-line tie-breaking | ☐ |
| DSA | Grids: BFS vs DFS, mark on enqueue, recursion depth, multi-source BFS | ☐ |

**Action rules:**

- Any **≤ 2** → redo that day's code example from a blank file before Day 29.
- Any **3** → add it to `GAPS.md` with a re-test date on Day 35.
- All **≥ 4** → stretch: LC 295 (Find Median from Data Stream) and LC 253 (Meeting Rooms II) this weekend.

**Retro (write 3 lines in `GAPS.md`):**

```text
Most surprising thing this week:
One production incident I now understand better:
One thing I'll do differently in Week 5:
```

---

[← Day 27 — Grid BFS/DFS](day-27-graph-grid-traversal.md) · [Week 4 overview](README.md) · [Week 5 — Day 29 →](../week-05/day-29-import-system-packaging.md)
