# Day 24 — Backpressure & Dynamic Micro-batching for Inference

[← Week 4](README.md) · **Track:** Python Systems · **Time box:** 60 min (concept 20 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** A GPU running one request at a time spends most of its time on fixed per-call overhead: kernel launches, reading the weights from memory, Python dispatch. Batching 16 requests often costs barely more than running 1. Inference servers (Triton, vLLM, TGI, Ray Serve) all use **dynamic batching**, and they're all protected by **bounded queues** so a traffic spike becomes fast `503`s instead of an out-of-memory kill.

**Prerequisites:** Day 5 (futures and tasks) and Day 22 (cancellation, timeouts).

---

## 1. Core Concept

### 1.1 Backpressure: the queue bound is the contract

When a producer outpaces a consumer, the gap has to go *somewhere*:

| Strategy | What happens to excess work | Failure mode |
|---|---|---|
| **Unbounded queue** | Piles up in memory | Latency grows without limit, then the OOM killer. Every queued item is already stale |
| **Bounded queue, blocking put** | The producer slows to the consumer's pace | Correct *inside* a pipeline. Pushes pressure upstream |
| **Bounded queue, non-blocking put → reject** | Rejected immediately (**load shedding**) | Correct *at the edge*: the client gets `503` + `Retry-After` and can retry elsewhere |
| **Drop oldest / sample** | Discarded | Fine for metrics and telemetry, wrong for requests |

`asyncio.Queue(maxsize=N)` gives you both bounded variants. `await q.put(x)` suspends while the queue is full, which is backpressure. `q.put_nowait(x)` raises `asyncio.QueueFull`, which you use for shedding. Internally the queue keeps a `deque` of items plus two `deque`s of waiter futures (`_getters`, `_putters`). `get()` wakes one putter and `put()` wakes one getter, so it's a direct application of Day 5's park-on-a-future mechanism.

**Sizing the bound:** by Little's law, items in the system = arrival rate × time in system. If the model sustains 400 req/s and your latency SLO allows 250 ms of queueing, a queue deeper than about 100 items guarantees SLO violations for everything behind it. **Bound by the latency budget, not by memory.**

### 1.2 Shutting down a producer/consumer pipeline

Consumers block forever in `await q.get()`, so they need an explicit "no more work" signal:

- **Sentinel (any version):** after all producers finish, put one `STOP` object per consumer. Each consumer exits when it receives one. Combine this with `q.task_done()` + `await q.join()` if you need "everything processed" semantics.
- **`Queue.shutdown()` (3.13+):** `q.shutdown()` makes further `put()` raise `QueueShutDown`. Once the queue is drained, blocked and future `get()` calls raise `QueueShutDown` too. `q.shutdown(immediate=True)` discards the remaining items. It's cleaner than sentinels, but it requires 3.13.

```python
# 3.13+ consumer shape (not runnable on 3.12)
async def consumer(q: asyncio.Queue[Job]) -> None:
    while True:
        try:
            job = await q.get()
        except asyncio.QueueShutDown:
            return
        try:
            await process(job)
        finally:
            q.task_done()
```

### 1.3 Dynamic micro-batching

Each request becomes a `(payload, Future)` pair placed on a bounded queue. One **batcher task** loops:

```text
1. first = await queue.get()                    # sleep until there is ANY work
2. deadline = now + max_wait                    # bound the extra latency we add
3. keep taking items until len(batch) == max_batch_size OR deadline passes
4. drop items whose future is already done      # the caller timed out or disconnected
5. results = await model(batch)                 # one forward pass
6. future.set_result(r) for each request        # or set_exception(e) for all of them on failure
```

The two knobs trade latency against throughput:

| Knob | Too small | Too large |
|---|---|---|
| `max_batch_size` | GPU underused, low throughput | Per-batch latency grows; GPU memory (activations, KV cache) blows up |
| `max_wait_ms` | Batches stay small under light load | Every request at low traffic waits the full window |

Under heavy load batches fill instantly, so `max_wait` costs nothing. Under light load `max_wait` caps the added latency. That self-tuning is what makes it **dynamic**. Typical starting values for online inference are `max_wait` of 2–10 ms and `max_batch_size` sized from a GPU memory profile.

**Futures are the hand-off.** The HTTP handler awaits *its own* future, the batcher resolves it, and the event loop wakes the handler. With Day 22's `timeout` around `await fut`, a timed-out caller **cancels its future**. The batcher must check `fut.done()` before computing (to skip wasted GPU work) and before calling `set_result` (otherwise `InvalidStateError`).

---

## 2. Production Gotcha: the unbounded queue that turns a spike into an outage

```python
queue: asyncio.Queue[Request] = asyncio.Queue()       # 🚨 maxsize=0 means unbounded

@app.post("/generate")
async def generate(req: Request):
    fut = loop.create_future()
    await queue.put((req, fut))                       # never blocks, never rejects
    return await fut                                  # 🚨 no timeout
```

A traffic spike at 3× capacity looks fine for the first few seconds. Then:

1. The queue grows by 2× capacity per second. Every request waits behind all the earlier ones, so **latency climbs linearly**, eventually past every client timeout.
2. Clients time out and **retry**, which adds even more load (a retry storm). Their original requests are still in the queue, so the GPU spends its time computing **answers nobody is waiting for**.
3. Memory grows with queued payloads (prompts, images, tensors) until the pod is OOM-killed, losing *everything* in flight. The other replicas take the load and fall over one by one.

**Fixes, all of which are in the code below:**

- `asyncio.Queue(maxsize=N)` with `put_nowait` → `QueueFull` → `503` + `Retry-After`. Fast rejection is a feature.
- A deadline around `await fut` on the caller side, and a `fut.done()` filter on the batcher side to skip abandoned work.
- When the batcher stops, **fail every pending future**, so no caller awaits a future that will never resolve.
- Export the queue depth and batch size as metrics, and autoscale on queue depth or latency rather than CPU. GPU services often run at low CPU while being completely saturated.

---

## 3. Runnable Code Example

Save as `day24_batching.py`. Part 1 compares a bounded and an unbounded pipeline. Part 2 runs a micro-batcher against a fake model with a fixed per-call overhead, comparing no batching, waves of traffic, a lone request, and an overload burst.

```python
"""
Day 24 - Backpressure, pipeline shutdown, and dynamic micro-batching for inference.

Run:  python day24_batching.py        (Python 3.12+; Queue.shutdown() needs 3.13)
"""
from __future__ import annotations

import asyncio
import statistics
import time
from collections.abc import Awaitable, Callable, Sequence
from dataclasses import dataclass


# ---------------------------------------------------------------------------
# Part 1: bounded vs unbounded queue between a fast producer and slow consumers
# ---------------------------------------------------------------------------
class _Stop:
    """Sentinel type: one instance per consumer signals 'no more work'."""


STOP = _Stop()


async def producer(q: asyncio.Queue[int | _Stop], n: int) -> float:
    blocked = 0.0
    for i in range(n):
        t = time.perf_counter()
        await q.put(i)                 # suspends while the queue is full -> backpressure
        blocked += time.perf_counter() - t
    return blocked


async def consumer(q: asyncio.Queue[int | _Stop], depth_samples: list[int]) -> int:
    handled = 0
    while True:
        depth_samples.append(q.qsize())
        item = await q.get()
        try:
            if isinstance(item, _Stop):
                return handled
            await asyncio.sleep(0.02)  # slow downstream: DB write, model call, ...
            handled += 1
        finally:
            q.task_done()


async def run_pipeline(maxsize: int, n: int = 40, workers: int = 2) -> None:
    q: asyncio.Queue[int | _Stop] = asyncio.Queue(maxsize=maxsize)
    depths: list[int] = []
    start = time.perf_counter()
    async with asyncio.TaskGroup() as tg:
        consumers = [tg.create_task(consumer(q, depths)) for _ in range(workers)]
        blocked = await producer(q, n)
        for _ in range(workers):
            await q.put(STOP)          # sentinel shutdown: one per consumer
    label = f"maxsize={maxsize}" if maxsize else "unbounded"
    print(f"  {label:<11} handled={sum(c.result() for c in consumers)} "
          f"max_depth={max(depths):>2}  producer_blocked={blocked:.2f}s  "
          f"total={time.perf_counter() - start:.2f}s")


# ---------------------------------------------------------------------------
# Part 2: dynamic micro-batcher with load shedding
# ---------------------------------------------------------------------------
class Overloaded(Exception):
    """Queue full or shutting down: map to HTTP 503 + Retry-After."""


@dataclass(slots=True)
class _Pending:
    prompt: str
    future: asyncio.Future[int]


class MicroBatcher:
    def __init__(
        self,
        infer: Callable[[Sequence[str]], Awaitable[list[int]]],
        *,
        max_batch_size: int,
        max_wait_s: float,
        max_queue: int,
    ) -> None:
        self._infer = infer
        self._max_batch = max_batch_size
        self._max_wait = max_wait_s
        self._queue: asyncio.Queue[_Pending] = asyncio.Queue(maxsize=max_queue)
        self.batch_sizes: list[int] = []

    async def submit(self, prompt: str, timeout_s: float = 2.0) -> int:
        fut: asyncio.Future[int] = asyncio.get_running_loop().create_future()
        try:
            self._queue.put_nowait(_Pending(prompt, fut))
        except asyncio.QueueFull:
            raise Overloaded("queue full") from None       # shed load at the edge
        async with asyncio.timeout(timeout_s):              # on timeout, fut gets cancelled
            return await fut

    async def _collect(self) -> list[_Pending]:
        batch = [await self._queue.get()]                   # sleep until there is work
        deadline = asyncio.get_running_loop().time() + self._max_wait
        try:
            async with asyncio.timeout_at(deadline):
                while len(batch) < self._max_batch:
                    batch.append(await self._queue.get())   # returns at once if items queued
        except TimeoutError:
            pass                                            # window closed: ship what we have
        return batch

    async def run(self) -> None:
        live: list[_Pending] = []
        try:
            while True:
                batch = await self._collect()
                live = [p for p in batch if not p.future.done()]   # skip abandoned requests
                if not live:
                    continue
                self.batch_sizes.append(len(live))
                try:
                    results = await self._infer([p.prompt for p in live])
                except Exception as exc:                    # one failure fails the whole batch
                    for p in live:
                        if not p.future.done():
                            p.future.set_exception(exc)
                else:
                    for p, result in zip(live, results, strict=True):
                        if not p.future.done():             # caller may have timed out meanwhile
                            p.future.set_result(result)
        finally:
            # Shutdown: never leave a caller awaiting a future nobody will resolve.
            while not self._queue.empty():
                live.append(self._queue.get_nowait())
            for p in live:
                if not p.future.done():
                    p.future.set_exception(Overloaded("shutting down"))


async def fake_model(prompts: Sequence[str]) -> list[int]:
    """Fixed per-call overhead dominates, like a GPU forward pass."""
    await asyncio.sleep(0.03 + 0.002 * len(prompts))
    return [len(p) for p in prompts]


async def timed_submit(b: MicroBatcher, prompt: str, latencies: list[float]) -> int:
    t = time.perf_counter()
    result = await b.submit(prompt)
    latencies.append(time.perf_counter() - t)
    return result


async def scenario(name: str, max_batch: int, waves: list[int], max_queue: int = 64) -> None:
    batcher = MicroBatcher(fake_model, max_batch_size=max_batch, max_wait_s=0.02,
                           max_queue=max_queue)
    runner = asyncio.create_task(batcher.run())
    latencies: list[float] = []
    ok = shed = 0
    start = time.perf_counter()
    for wave in waves:
        results = await asyncio.gather(
            *(timed_submit(batcher, f"prompt-{i}", latencies) for i in range(wave)),
            return_exceptions=True,
        )
        ok += sum(not isinstance(r, BaseException) for r in results)
        shed += sum(isinstance(r, Overloaded) for r in results)
        await asyncio.sleep(0.05)                           # gap between traffic waves
    elapsed = time.perf_counter() - start
    runner.cancel()
    await asyncio.gather(runner, return_exceptions=True)
    p50 = statistics.median(latencies) * 1000
    print(f"  {name:<22} ok={ok:<3} shed={shed:<3} batches={len(batcher.batch_sizes):<3} "
          f"sizes={batcher.batch_sizes[:6]}{'...' if len(batcher.batch_sizes) > 6 else ''}")
    print(f"  {'':<22} p50={p50:.0f}ms max={max(latencies) * 1000:.0f}ms total={elapsed:.2f}s")


async def main() -> None:
    print("== Part 1: backpressure")
    await run_pipeline(maxsize=0)
    await run_pipeline(maxsize=5)

    print("\n== Part 2: micro-batching (3 waves of 20 requests)")
    await scenario("no batching (size=1)", max_batch=1, waves=[20, 20, 20])
    await scenario("dynamic (size=8)", max_batch=8, waves=[20, 20, 20])
    await scenario("lone request", max_batch=8, waves=[1])
    await scenario("overload burst", max_batch=8, waves=[100], max_queue=32)


if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output (timings vary by machine and OS timer resolution):**

```text
== Part 1: backpressure
  unbounded   handled=40 max_depth=42  producer_blocked=0.00s  total=0.63s
  maxsize=5   handled=40 max_depth= 5  producer_blocked=0.54s  total=0.64s

== Part 2: micro-batching (3 waves of 20 requests)
  no batching (size=1)   ok=60  shed=0   batches=60  sizes=[1, 1, 1, 1, 1, 1]...
                         p50=499ms max=963ms total=3.06s
  dynamic (size=8)       ok=60  shed=0   batches=9   sizes=[8, 8, 4, 8, 8, 4]...
                         p50=94ms max=174ms total=0.71s
  lone request           ok=1   shed=0   batches=1   sizes=[1]
                         p50=79ms max=79ms total=0.14s
  overload burst         ok=32  shed=68  batches=4   sizes=[8, 8, 8, 8]
                         p50=119ms max=191ms total=0.25s
```

**How to read it:**

- **Part 1:** the unbounded queue absorbed everything immediately (max depth 42 = 40 items + 2 sentinels, producer never blocked). In a real service that's 40 stale requests sitting in memory. With `maxsize=5`, depth never exceeded 5, and the producer spent most of its time suspended, running at the consumers' pace.
- **No batching vs dynamic:** same 60 requests, and the batched version was over 4× faster with a p50 about 5× lower, because the fixed 30 ms model overhead is paid 9 times instead of 60. Each wave of 20 splits into batches of `[8, 8, 4]`. The trailing 4 waited out the 20 ms window and then shipped anyway.
- **Lone request:** paid the full `max_wait` window (20 ms) plus one model call (about 32 ms). On Windows, the event loop's ~15 ms timer granularity pushes the total higher. That's the latency `max_wait` costs at low traffic.
- **Overload burst:** exactly 32 accepted (the queue bound) and 68 rejected instantly. Those 68 clients learn within microseconds that they should retry elsewhere, instead of timing out after seconds.

**Exercises (10 min):**

1. Add a 50 ms timeout in `submit` for the overload scenario. Count how many accepted requests now time out, and confirm the batcher skips their futures (`batch_sizes` shrinks).
2. Make `fake_model` raise on batches containing `"prompt-13"`. Check that every caller in that batch, and only that batch, gets the exception.
3. On 3.13+, rewrite Part 1 with `q.shutdown()` instead of sentinels.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `dynamic batching inference server explained` / `continuous batching LLM inference` | **Anyscale / Ray Summit**, **NVIDIA Developer** | How Triton/Ray Serve batch requests, and why LLM servers moved to continuous (iteration-level) batching |
| Concepts | `Little's law queueing latency backpressure` | Conference talks (e.g. **Strange Loop**, **QCon / InfoQ**) | Queueing theory intuition: why latency explodes near saturation |
| Python | `asyncio Queue producer consumer` | **ArjanCodes**, **mCoding**, **Corey Schafer** | Clean producer/consumer patterns in asyncio |

**Reading:** Marc Brooker's blog posts on load shedding and retries, and the AWS Builders' Library articles "Using load shedding to avoid overload" and "Timeouts, retries, and backoff with jitter". These are widely referenced and directly applicable.

---

[← Day 23 — Heaps / Top-K](day-23-heaps-top-k.md) · [Day 25 — Intervals →](day-25-intervals.md)
