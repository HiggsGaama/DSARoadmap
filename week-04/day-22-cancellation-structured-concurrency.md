# Day 22 — Cancellation & Structured Concurrency

[← Week 4](README.md) · **Track:** Python Systems · **Time box:** 60 min (concept 20 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** Every timeout, client disconnect, deploy (`SIGTERM`) and failed sibling request in an async service shows up *as a cancellation*. If cancellation is handled wrong, timeouts silently stop working, rolling deploys drop in-flight requests, and shutdowns hang until Kubernetes sends `SIGKILL`.

**Prerequisite:** Day 5 (how a `Task` drives a coroutine with `send()`).

---

## 1. Core Concept

### 1.1 `task.cancel()` is a request, not an interrupt

Recall from Day 5 that a `Task` advances its coroutine with `coro.send(None)` each time the future it's parked on resolves. `task.cancel()` does **not** stop anything immediately. It:

1. sets an internal "must cancel" flag and increments the task's **cancellation counter** (`task.cancelling()`, 3.11+);
2. if the task is parked on a future, cancels that future, which schedules the task to wake up;
3. on the task's next step, calls `coro.throw(CancelledError())` instead of `coro.send(None)`.

So the exception is raised **at the `await` where the coroutine is currently suspended**. Because the loop is single-threaded, whatever *requests* the cancellation (a timeout's timer, another task, a signal handler) can only run while your coroutine is suspended. Sync code between awaits therefore delays both the request and its delivery. A 2-second CPU loop ignores cancellations and timeouts for 2 seconds. A task cancelled *before it ever ran* gets `CancelledError` at its very first step, so its body never executes at all, including any `try/finally` written inside it.

### 1.2 `CancelledError` is a `BaseException` (3.8+)

Since 3.8, `asyncio.CancelledError` subclasses `BaseException`, not `Exception`. That's deliberate: the very common `except Exception:` (log and continue) no longer accidentally swallows cancellation. It still gets swallowed by:

- bare `except:` and `except BaseException:`;
- `except asyncio.CancelledError:` that **doesn't re-raise**;
- `contextlib.suppress(BaseException)`;
- `return` or `break` inside a `finally:` block (this discards the in-flight exception; 3.14 added a `SyntaxWarning` for it, PEP 765).

**Rule:** you may *catch* `CancelledError` to do cleanup, but you must re-raise it. If cleanup needs to `await` (flush a buffer, release a pooled connection), that's allowed inside `except`/`finally`. Keep it short and bounded, because the canceller is waiting.

### 1.3 How `asyncio.timeout()` turns cancellation into `TimeoutError`

`asyncio.timeout(delay)` (3.11+) is built entirely on cancellation:

```text
__aenter__: remember current task; schedule loop.call_at(deadline, _on_timeout)
_on_timeout: task.cancel()                 # a plain cancellation request (counter += 1)
__aexit__(exc):
    if timeout fired and exc is CancelledError and task.uncancel() <= count_at_entry:
        raise TimeoutError from exc        # this cancellation was OURS -> convert it
    else: propagate as-is                  # someone ELSE cancelled us -> stay cancelled
```

`uncancel()` decrements the counter. Comparing it with the value recorded on entry is how nested timeouts and `TaskGroup`s tell **their own** cancellation apart from an **external** one, such as a server shutdown cancelling a request that happens to be inside a timeout block. `asyncio.wait_for` has used the same mechanism since 3.12.

Consequence: if the code inside swallows the `CancelledError`, `__aexit__` sees a normal exit, **no `TimeoutError` is raised**, and the caller believes the call succeeded, possibly long after the deadline.

### 1.4 `asyncio.shield()`: protecting a critical section

`await asyncio.shield(aw)` wraps `aw` in an outer future. Cancelling the *awaiting* task cancels only the outer future. The inner task keeps running. Use it for operations that must finish once started (committing a payment, writing an idempotency record). Two caveats:

- The caller still gets `CancelledError` (or `TimeoutError`), so it must not assume the work failed.
- Keep a **strong reference** to the inner task (Day 5: the loop keeps only weak ones). A shielded task nobody references can still be garbage-collected.

### 1.5 Structured concurrency with `TaskGroup` (3.11+)

`async with asyncio.TaskGroup() as tg:` guarantees **no child outlives the block**:

- If a child raises (anything except `CancelledError`), the group cancels every other child and the body of the `async with` itself, waits for all of them to finish (including their `finally` blocks), then raises an `ExceptionGroup` containing all the non-cancellation errors. Handle it with `except* ValueError:` (Day 8).
- If the *parent* task is cancelled, all children are cancelled too, and the parent sees `CancelledError`, not an `ExceptionGroup`.
- Compare `asyncio.gather(...)`: by default it **does not cancel siblings** when one fails. The failure propagates, and the other tasks keep running unsupervised. `return_exceptions=True` collects errors but still never cancels anything.

### 1.6 Graceful shutdown

Kubernetes sends `SIGTERM`, waits `terminationGracePeriodSeconds` (default 30 s), then sends `SIGKILL`. A correct async server:

1. Converts `SIGTERM` into an event: `loop.add_signal_handler(SIGTERM, stop.set)`. This is POSIX-only; on Windows it raises `NotImplementedError`. Never do real work inside a raw `signal.signal` handler.
2. Stops accepting new work (and fails readiness probes so the load balancer drains it).
3. Lets in-flight work finish, bounded by a timeout **shorter than** the grace period.
4. Cancels whatever is left, then exits. `asyncio.run()` cancels remaining tasks on exit anyway, but by then you've lost control over ordering and logging.

---

## 2. Production Gotcha: swallowing cancellation, plus cancelled-but-still-running threads

### 2a. The "resilient" fallback that disables your timeouts

```python
async def get_recommendations(user_id: str) -> list[Item]:
    try:
        return await reco_client.fetch(user_id)
    except asyncio.CancelledError:          # 🚨 "if it's cancelled, just serve defaults"
        return DEFAULT_ITEMS
```

The author meant "degrade gracefully". In practice:

- `async with asyncio.timeout(0.3): await get_recommendations(...)` **never raises `TimeoutError`**, so the caller's timeout metrics read 0% while users wait.
- On shutdown, `task.cancel()` is ignored. The task returns normally and keeps going, so the process hangs until `SIGKILL`, dropping other in-flight requests.
- Inside a `TaskGroup`, a child that ignores cancellation blocks the whole group from unwinding.

**Fix:** put the fallback on `TimeoutError` *at the call site that owns the deadline*, and let `CancelledError` propagate:

```python
try:
    async with asyncio.timeout(0.3):
        items = await reco_client.fetch(user_id)
except TimeoutError:
    items = DEFAULT_ITEMS
```

Linters help: ruff's `ASYNC` rules and `BLE001` (blind except) flag several of these patterns.

### 2b. Cancelling `to_thread` doesn't stop the thread

`await asyncio.to_thread(fn)` hands `fn` to a thread pool. Cancelling or timing out the await only abandons the *result*. Python threads can't be killed, so `fn` runs to completion in the background. If `fn` writes to a database or charges a card, it **still happens** after the caller reported a timeout. Make such functions idempotent, or pass them a `threading.Event` they check cooperatively.

---

## 3. Runnable Code Example

Save as `day22_cancellation.py`. It demonstrates delivery timing, the swallowing gotcha, the thread gotcha, `shield`, `TaskGroup` propagation and a graceful-shutdown skeleton. On Windows the `SIGTERM` handler isn't supported, so the script simulates the signal.

```python
"""
Day 22 - Cancellation done right: delivery, swallowing, shield, timeouts, TaskGroup, shutdown.

Run:  python day22_cancellation.py        (Python 3.12+)
"""
from __future__ import annotations

import asyncio
import signal
import time


# ---------------------------------------------------------------------------
# 1. cancel() is a REQUEST, delivered at the next suspension point
# ---------------------------------------------------------------------------
def busy(seconds: float) -> None:
    deadline = time.perf_counter() + seconds
    while time.perf_counter() < deadline:        # sync work: nothing can interrupt this
        pass


async def chunked_job(t0: float, log: list[str]) -> None:
    try:
        for i in range(5):
            busy(0.05)
            log.append(f"t={time.perf_counter() - t0:.2f}s chunk {i} done")
            await asyncio.sleep(0)               # checkpoint: the only place cancel can land
    except asyncio.CancelledError:
        log.append(f"t={time.perf_counter() - t0:.2f}s CancelledError delivered at checkpoint")
        raise                                    # always re-raise


def slow_sync_write(log: list[str]) -> None:
    time.sleep(0.2)
    log.append("thread finished its write anyway")


async def demo_delivery() -> None:
    log: list[str] = []
    t0 = time.perf_counter()
    task = asyncio.create_task(chunked_job(t0, log))

    def request_cancel() -> None:
        log.append(f"t={time.perf_counter() - t0:.2f}s cancel() requested (scheduled for t=0.07s)")
        task.cancel()

    asyncio.get_running_loop().call_later(0.07, request_cancel)
    try:
        await task
    except asyncio.CancelledError:
        pass
    for line in log:
        print("  " + line)
    print("  task.cancelled() =", task.cancelled())

    # Cancelling an await on to_thread() does NOT stop the thread.
    thread_log: list[str] = []
    try:
        async with asyncio.timeout(0.05):
            await asyncio.to_thread(slow_sync_write, thread_log)
    except TimeoutError:
        print("  to_thread call timed out; thread_log =", thread_log)
    await asyncio.sleep(0.25)
    print("  250 ms later;          thread_log =", thread_log)


# ---------------------------------------------------------------------------
# 2. GOTCHA: swallowing CancelledError breaks timeouts silently
# ---------------------------------------------------------------------------
async def fetch_swallowing() -> str:
    try:
        await asyncio.sleep(1.0)                 # pretend: slow upstream call
        return "fresh data"
    except asyncio.CancelledError:               # BUG: eats the cancellation
        return "fallback data"


async def fetch_correct() -> str:
    try:
        await asyncio.sleep(1.0)
        return "fresh data"
    except asyncio.CancelledError:
        # quick, bounded bookkeeping is fine here -- then propagate
        raise


async def demo_swallow() -> None:
    for fn in (fetch_swallowing, fetch_correct):
        start = time.perf_counter()
        try:
            async with asyncio.timeout(0.1):
                result = await fn()
            outcome = f"returned {result!r} -> caller sees success, timeout lost"
        except TimeoutError:
            outcome = "TimeoutError raised -> caller can react"
        print(f"  {fn.__name__:<17} {outcome}  [{time.perf_counter() - start:.2f}s]")


# ---------------------------------------------------------------------------
# 3. shield(): protect a critical section from an OUTER cancellation
# ---------------------------------------------------------------------------
async def commit_payment(log: list[str]) -> str:
    await asyncio.sleep(0.15)
    log.append("payment committed")
    return "txn-42"


async def demo_shield() -> None:
    log: list[str] = []
    inner = asyncio.create_task(commit_payment(log))   # keep a strong reference!
    try:
        async with asyncio.timeout(0.05):
            await asyncio.shield(inner)
    except TimeoutError:
        print("  caller timed out after 50 ms; inner task done?", inner.done())
    print("  inner result once finished:", await inner, "| log:", log)


# ---------------------------------------------------------------------------
# 4. TaskGroup: first failure cancels siblings; errors arrive as ExceptionGroup
# ---------------------------------------------------------------------------
async def child(name: str, delay: float, fail: bool, log: list[str]) -> None:
    try:
        await asyncio.sleep(delay)
        if fail:
            raise ValueError(f"{name} failed")
        log.append(f"{name} ok")
    except asyncio.CancelledError:
        log.append(f"{name} cancelled")
        raise
    finally:
        await asyncio.sleep(0)                   # await in finally is legal: async cleanup
        log.append(f"{name} cleanup")


async def demo_taskgroup() -> None:
    log: list[str] = []
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(child("fast", 0.01, False, log))
            tg.create_task(child("bad", 0.05, True, log))
            tg.create_task(child("slow", 1.00, False, log))
    except* ValueError as eg:
        print("  caught:", [str(e) for e in eg.exceptions])
    print("  log:", log)


# ---------------------------------------------------------------------------
# 5. Graceful shutdown on SIGTERM (Kubernetes: SIGTERM, grace period, then SIGKILL)
# ---------------------------------------------------------------------------
async def serve_forever(stop: asyncio.Event, log: list[str]) -> None:
    async def handle(req_id: int) -> None:
        await asyncio.sleep(0.05)
        log.append(f"req-{req_id} done")

    req_id = 0
    async with asyncio.TaskGroup() as tg:        # in-flight requests live here
        while not stop.is_set():
            tg.create_task(handle(req_id))
            req_id += 1
            await asyncio.sleep(0.02)            # stand-in for "await the next connection"
        log.append(f"stop requested after {req_id} requests; draining in-flight")
    log.append("all in-flight requests drained")  # TaskGroup exit waited for them


async def demo_shutdown() -> None:
    loop = asyncio.get_running_loop()
    stop = asyncio.Event()
    try:
        loop.add_signal_handler(signal.SIGTERM, stop.set)   # POSIX event loops only
        print("  SIGTERM handler installed")
    except NotImplementedError:
        print("  add_signal_handler unsupported (Windows); simulating SIGTERM")
    loop.call_later(0.1, stop.set)                          # simulate the signal for the demo
    log: list[str] = []
    async with asyncio.timeout(5):                          # hard cap < terminationGracePeriod
        await serve_forever(stop, log)
    for line in log[-4:]:
        print("  " + line)


async def main() -> None:
    print("== 1. delivery")
    await demo_delivery()
    print("== 2. swallowing")
    await demo_swallow()
    print("== 3. shield")
    await demo_shield()
    print("== 4. TaskGroup")
    await demo_taskgroup()
    print("== 5. shutdown")
    await demo_shutdown()


if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output (timings vary slightly):**

```text
== 1. delivery
  t=0.05s chunk 0 done
  t=0.10s chunk 1 done
  t=0.15s chunk 2 done
  t=0.15s cancel() requested (scheduled for t=0.07s)
  t=0.15s CancelledError delivered at checkpoint
  task.cancelled() = True
  to_thread call timed out; thread_log = []
  250 ms later;          thread_log = ['thread finished its write anyway']
== 2. swallowing
  fetch_swallowing  returned 'fallback data' -> caller sees success, timeout lost  [0.11s]
  fetch_correct     TimeoutError raised -> caller can react  [0.11s]
== 3. shield
  caller timed out after 50 ms; inner task done? False
  inner result once finished: txn-42 | log: ['payment committed']
== 4. TaskGroup
  caught: ['bad failed']
  log: ['fast ok', 'fast cleanup', 'bad cleanup', 'slow cancelled', 'slow cleanup']
== 5. shutdown
  add_signal_handler unsupported (Windows); simulating SIGTERM
  req-1 done
  stop requested after 3 requests; draining in-flight
  req-2 done
  all in-flight requests drained
```

Notice in block 1: the cancel timer was due at 0.07 s, but the loop was busy running sync chunks and couldn't fire it until about 0.15 s. The `CancelledError` was then thrown at the checkpoint where the task was parked. A real timeout behaves the same way, so a slow sync section makes every deadline in the process late. In block 4, `slow` was cancelled and its `finally` still ran before the `ExceptionGroup` reached the caller.

**Exercises (10 min):**

1. Remove the `await asyncio.sleep(0)` checkpoint from `chunked_job`. How many chunks run, and is the task even cancelled? (`cancel()` on a finished task returns `False`.)
2. Replace `asyncio.TaskGroup()` in `demo_taskgroup` with `asyncio.gather(...)`. Does `slow` still get cancelled? (It doesn't, and it keeps running after the exception propagates.)
3. Inside `child`, change `raise` in the `except CancelledError` branch to `return`. Run block 4 and explain the result in terms of `cancelling()` / `uncancel()`.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `Lukasz Langa import asyncio cancellation` | **EdgeDB** channel (Łukasz Langa) | The `import asyncio` series covers cancellation and task lifecycles from the core-dev side |
| Deep dive | `Nathaniel Smith structured concurrency trio` | **PyCon / conference recordings** (Nathaniel J. Smith) | Where `TaskGroup` comes from: the "nursery" model and why unscoped tasks are the async `goto` |
| Practical | `anthonywritescode asyncio TaskGroup` / `mCoding asyncio` | **anthonywritescode**, **mCoding** | Short, concrete demos of `TaskGroup`, `timeout` and `except*` |

**Reading:** the "Task Cancellation" and "Task Groups" sections of the official `asyncio-task` docs are short and precise. Read them once end to end.

---

[← Day 21 — Consolidation](../week-03/day-21-consolidation.md) · [Day 23 — Heaps / Top-K →](day-23-heaps-top-k.md)
