# Day 5 — asyncio Internals: The Event Loop, Tasks, and What "Blocking" Really Means

[← Week 1](README.md) · **Track:** Python Systems · **Time box:** 60 min (concept 20 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** FastAPI, aiohttp, LLM gateways, vector-DB clients and streaming inference proxies all run on one thread executing one event loop. If a single synchronous call slips in, *every* request in that process stalls, and the p99 latency graph doesn't tell you why.

**Prerequisite:** Day 3, §1.3 (`yield from` as a bidirectional proxy).

---

## 1. Core Concept

### 1.1 The loop is a single-threaded callback scheduler

Strip away the API surface and `BaseEventLoop` holds three things:

| Structure | Contents |
|---|---|
| `_ready` | A `deque` of `Handle`s: callbacks to run **on this iteration** |
| `_scheduled` | A heap of `TimerHandle`s ordered by deadline (`call_later`, `asyncio.sleep`) |
| `_selector` | An OS readiness API: `epoll` (Linux), `kqueue` (macOS/BSD). Windows defaults to the **Proactor** loop on IOCP (completion-based, not readiness-based) |

**One iteration (`_run_once`)**, simplified:

```text
1. timeout = 0 if _ready else (nearest timer deadline − now)     # sleep only if nothing to do
2. events  = selector.select(timeout)                             # the ONLY place the loop waits
3. for each ready fd: append its callback to _ready
4. move every expired timer from _scheduled to _ready
5. n = len(_ready); run exactly n callbacks                       # callbacks queued during this step wait
```

Step 5 is where *your* code runs. There's no preemption: a callback runs until it returns. While it runs, nothing else in the process makes progress. No other request is served, no timer fires, no socket is read, no health check is answered.

### 1.2 Tasks drive coroutines with `send()`

A coroutine is a generator-like object (Day 3). A `Task` wraps one and drives it:

```text
Task.__step():
    result = coro.send(None)          # run the coroutine until it suspends
    ├─ raises StopIteration(v)  → task.set_result(v)              # finished
    ├─ raises exception e       → task.set_exception(e)
    └─ yields a Future f        → f.add_done_callback(task.__wakeup)   # park the task
                                  (the task now isn't in _ready at all; it costs nothing)

later: socket readable → protocol calls f.set_result(data)
       → f schedules its callbacks via loop.call_soon → task.__wakeup → task.__step() again
```

How does `await reader.read()` deep in your stack reach the Task? Each `await` delegates downward like `yield from`. At the bottom, `Future.__await__` does `yield self`, and that object travels **up through every frame** to `Task.__step`, which parks the task on it. When the future resolves, `send()` resumes the whole chain exactly where it left off.

### 1.3 Two consequences most people miss

1. **`await` doesn't always yield to the loop.** Control returns to the loop only when something at the bottom of the chain yields a pending `Future`. This coroutine never suspends on a cache hit:

   ```python
   async def get(key: str) -> bytes:
       if key in cache:
           return cache[key]          # no suspension
       return await fetch(key)
   ```

   A loop over 1M cache hits like `for k in keys: await get(k)` monopolizes the loop for the whole duration. Insert `await asyncio.sleep(0)` periodically. It's special-cased as a bare `yield` that gives control back for exactly one iteration.

2. **The loop holds only weak references to tasks.** `asyncio.create_task()` returns the only strong reference to the task. Drop it, and a task suspended on a future that nobody else references can be **garbage-collected mid-flight**, so its work silently never completes. The asyncio docs warn about this explicitly. Exceptions in tasks nobody awaits are logged only at GC time ("Task exception was never retrieved"), if at all.

### 1.4 Lifecycle and where threads come in

- `asyncio.run(main())` creates a fresh loop, runs `main` to completion, **cancels all remaining tasks**, finalizes async generators (`shutdown_asyncgens`), shuts down the default executor, and closes the loop.
- `asyncio.to_thread(fn, *args)` (3.9+) runs `fn` in the loop's default `ThreadPoolExecutor` and awaits the result through a future. The GIL is released during blocking I/O and `time.sleep`, so this is fine for blocking *I/O*. For CPU-bound pure-Python work, threads still contend for the GIL, so use a `ProcessPoolExecutor` via `loop.run_in_executor`.
- `TaskGroup` (3.11+) gives you structured concurrency: if any child raises, the siblings get cancelled, and the errors surface together as an `ExceptionGroup`. Prefer it over bare `gather`.

---

## 2. Production Gotcha: hidden blocking calls, plus fire-and-forget tasks that vanish

### 2a. The blocking call that serializes your whole service

```python
@app.post("/embed")
async def embed(req: EmbedRequest) -> EmbedResponse:
    vectors = openai_client.embeddings.create(...)   # 🚨 sync client inside async def
    meta = requests.get(METADATA_URL).json()          # 🚨 sync HTTP
    return EmbedResponse(vectors=..., meta=meta)
```

Each sync call parks the **entire event loop** for its full duration. At 200 ms per call and 50 concurrent requests, the last request waits about 10 seconds even though the CPU is idle. Common culprits include `requests`, sync SDKs (boto3, the sync OpenAI or Anthropic clients, sync Redis/Postgres drivers), `time.sleep`, large `json.loads` or `pydantic` validation on multi-MB payloads, file I/O on network filesystems, and CPU-heavy tokenization.

**Detect:**

- `asyncio.run(main(), debug=True)` or `PYTHONASYNCIODEBUG=1` logs `Executing <Task …> took 0.201 seconds` for any callback slower than `loop.slow_callback_duration` (default 100 ms). Don't leave this on in production, because debug mode adds overhead.
- In production, run a **loop-lag monitor** (in the code below) that reports the delay as a metric.
- `py-spy dump --pid <pid>` on a stalled worker shows exactly which sync call the loop thread is stuck in.

**Fix:** use native async clients (`httpx.AsyncClient`, `AsyncOpenAI`, `asyncpg`, `redis.asyncio`). Wrap unavoidable sync calls in `await asyncio.to_thread(...)`. Send CPU-bound work to a process pool or a separate worker. In FastAPI, a plain `def` endpoint (not `async def`) is automatically run in a threadpool, so declaring an endpoint `async def` and then calling sync code is the worst option.

### 2b. Fire-and-forget tasks that never finish

```python
async def handle(req):
    asyncio.create_task(emit_audit_log(req))   # 🚨 result discarded; task may be GC'd; errors vanish
    return ok()
```

**Fix:** keep strong references in a set, remove them in a done-callback, and log failures there. Or put the work under a `TaskGroup` or a proper queue. On shutdown, drain the set before the loop closes, because `asyncio.run` cancels whatever is still pending.

---

## 3. Runnable Code Example

Save as `day05_asyncio.py`. It runs the same batch twice, once with a hidden blocking call and once fixed, with a loop-lag monitor running and safe fire-and-forget audit events.

```python
"""
Day 5 — asyncio: bounded concurrency, offloading blocking calls, loop-lag detection,
and fire-and-forget tasks that can't vanish.

Run:  python day05_asyncio.py        (Python 3.11+ for TaskGroup / asyncio.timeout)
"""
from __future__ import annotations

import asyncio
import random
import time
from collections.abc import Coroutine
from dataclasses import dataclass
from typing import Any


@dataclass(frozen=True, slots=True)
class Result:
    job_id: int
    ok: bool
    detail: str


# ---------------------------------------------------------------------------
# Safe fire-and-forget: the event loop keeps only WEAK refs to tasks.
# ---------------------------------------------------------------------------
_background_tasks: set[asyncio.Task[Any]] = set()


def _log_failure(task: asyncio.Task[Any]) -> None:
    if not task.cancelled() and (exc := task.exception()) is not None:
        print(f"  [background] {task.get_name()} failed: {exc!r}")


def fire_and_forget(coro: Coroutine[Any, Any, Any], *, name: str) -> asyncio.Task[Any]:
    task = asyncio.create_task(coro, name=name)
    _background_tasks.add(task)                       # strong ref until done
    task.add_done_callback(_background_tasks.discard)
    task.add_done_callback(_log_failure)              # exceptions are retrieved and logged
    return task


async def emit_audit_event(job_id: int) -> None:
    await asyncio.sleep(0.01)                         # e.g. POST to an audit service
    if job_id == 7:
        raise ConnectionError("audit sink unavailable")


# ---------------------------------------------------------------------------
# Blocking work and loop-lag detection
# ---------------------------------------------------------------------------
def legacy_sync_call(job_id: int) -> str:
    """Stands in for requests.get(), a boto3 call, or a sync DB driver."""
    time.sleep(0.2)
    return f"payload-{job_id}"


async def loop_lag_monitor(interval: float = 0.05, threshold: float = 0.1) -> None:
    """If sleep(interval) takes much longer than `interval`, something blocked the loop.
    In production, export `lag` as a histogram metric instead of printing."""
    loop = asyncio.get_running_loop()
    while True:
        start = loop.time()
        await asyncio.sleep(interval)
        lag = loop.time() - start - interval
        if lag > threshold:
            print(f"  [lag-monitor] event loop blocked ~{lag * 1000:.0f} ms")


# ---------------------------------------------------------------------------
# The request handler
# ---------------------------------------------------------------------------
async def handle_job(job_id: int, sem: asyncio.Semaphore, *, blocking_bug: bool) -> Result:
    async with sem:  # bound in-flight work: protects downstream services and our memory
        try:
            async with asyncio.timeout(5.0):  # every outbound call needs a deadline
                await asyncio.sleep(random.uniform(0.05, 0.15))  # simulated async I/O
                if blocking_bug:
                    payload = legacy_sync_call(job_id)            # BUG: freezes the whole loop
                else:
                    payload = await asyncio.to_thread(legacy_sync_call, job_id)
        except TimeoutError:
            return Result(job_id, False, "timeout")
    fire_and_forget(emit_audit_event(job_id), name=f"audit-{job_id}")
    return Result(job_id, True, payload)


async def run_batch(n: int, concurrency: int, *, blocking_bug: bool) -> list[Result]:
    sem = asyncio.Semaphore(concurrency)
    async with asyncio.TaskGroup() as tg:  # structured: one failure cancels the siblings
        tasks = [tg.create_task(handle_job(i, sem, blocking_bug=blocking_bug)) for i in range(n)]
    return [t.result() for t in tasks]


async def main() -> None:
    for blocking_bug in (True, False):
        print(f"\n== {'BLOCKING (bug)' if blocking_bug else 'to_thread (fixed)'}")
        monitor = asyncio.create_task(loop_lag_monitor())
        start = time.perf_counter()
        results = await run_batch(10, concurrency=5, blocking_bug=blocking_bug)
        elapsed = time.perf_counter() - start

        # Graceful drain: let in-flight background work finish (asyncio.run() would
        # cancel it), and give the monitor a moment to report the final stall.
        await asyncio.gather(*_background_tasks, return_exceptions=True)
        await asyncio.sleep(0.1)
        monitor.cancel()
        print(f"  {sum(r.ok for r in results)}/{len(results)} ok in {elapsed:.2f}s")


if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output (timings vary):**

```text
== BLOCKING (bug)
  [lag-monitor] event loop blocked ~747 ms
  [lag-monitor] event loop blocked ~559 ms
  [lag-monitor] event loop blocked ~544 ms
  [background] audit-7 failed: ConnectionError('audit sink unavailable')
  10/10 ok in 2.06s

== to_thread (fixed)
  [background] audit-7 failed: ConnectionError('audit sink unavailable')
  10/10 ok in 0.63s
```

Ten jobs with five allowed concurrently should take two "waves" of about 0.3 s each. The buggy version takes more than 2 s because the five 200 ms sync calls in each wave run **back to back on the loop thread**. Even the semaphore can't help, because the sync call blocks everything, the semaphore included.

**Exercises (10 min):**

1. Run with `asyncio.run(main(), debug=True)` and find the `Executing <Task …> took 0.2xx seconds` warnings.
2. Remove `_background_tasks.add(task)` and the final `gather`. Watch the audit-7 error disappear or turn into "Task exception was never retrieved" at shutdown.
3. Set `asyncio.timeout(0.1)` and explain why the **buggy** path still finishes slowly: a timeout can only cancel a task at an `await`, and it can't interrupt a sync call already running.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `Lukasz Langa import asyncio series` | **EdgeDB** channel (Łukasz Langa, CPython core dev) | Multi-part series on how the loop, futures and tasks actually work |
| Deep dive | `David Beazley Build Your Own Async` | **PyCon India / conference recordings** | Live-codes an event loop from scratch: the best way to get rid of the "magic" |
| Practical | `mCoding async await python` / `ArjanCodes asyncio` | **mCoding**, **ArjanCodes** | Concise mental model; real-world structuring of async code |

---

[← Day 4 — Two Pointers](day-04-two-pointers.md) · [Day 6 — Sliding Window →](day-06-sliding-window.md)
