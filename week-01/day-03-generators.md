# Day 3 — Generators as Suspended-Frame State Machines

[← Week 1](README.md) · **Track:** Python Systems · **Time box:** 55 min (concept 15 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** Streaming ETL, log processing, paginated API clients, LLM token streaming and `async`/`await` itself are all built on generators. A generator that is suspended and never closed is how connection pools run dry and file descriptors leak in services that "can't possibly be leaking".

---

## 1. Core Concept

### 1.1 Calling a generator function runs nothing

```python
def gen():
    print("started")
    yield 1

g = gen()     # prints nothing: returns a generator object wrapping an unstarted frame
```

The compiler marks any function containing `yield` with a `GENERATOR` flag. Calling it allocates a **generator object that owns an execution frame**: the instruction pointer, local variables and value stack. Since 3.11 the frame is a lightweight `_PyInterpreterFrame` embedded directly in the generator object, and a full frame object (`g.gi_frame`) is created lazily only when something inspects it.

This is the key mental model: **a generator is a function whose stack frame lives on the heap instead of on the C stack**, so it can be paused and resumed later.

### 1.2 The state machine

```text
                 next()/send(None)                 yield
  GEN_CREATED ─────────────────────► GEN_RUNNING ─────────► GEN_SUSPENDED
       │                                ▲    │                    │
       │ close()                        │    │ return / raise     │ next()/send(v)/throw(e)
       ▼                                │    ▼                    │
   GEN_CLOSED ◄─────────────────────────┴── GEN_CLOSED ◄──────────┘
```

`inspect.getgeneratorstate(g)` returns one of these four states. The operations:

| Operation | Effect at the suspended `yield` | Notes |
|---|---|---|
| `next(g)` | Resume; the `yield` expression evaluates to `None` | Same as `g.send(None)` |
| `g.send(v)` | Resume; the `yield` expression evaluates to `v` | `TypeError` if `v is not None` and the generator hasn't started; you must **prime** it with `next()` first |
| `g.throw(exc)` | Raise `exc` *at the yield point* | The generator can catch it and keep going |
| `g.close()` | Raise `GeneratorExit` at the yield point | `finally:` and `with` blocks run. If the generator *yields again* instead of exiting, you get `RuntimeError: generator ignored GeneratorExit` |
| `return v` (inside) | Raises `StopIteration(v)` to the caller | `for` loops swallow it; `yield from` captures `v` |
| GC of a suspended generator | Calls `close()` | **Cleanup timing depends on when the GC runs** (see §2) |

**PEP 479 (3.7+):** if a `StopIteration` escapes from inside a generator body (for example, a bare `next(it)` on an exhausted iterator), it's converted to `RuntimeError`. Before PEP 479 it silently ended the generator early.

### 1.3 `yield from` is a bidirectional proxy, and the basis of `await`

`result = yield from sub` does much more than `for x in sub: yield x`:

- It forwards `next`, `send(v)`, `throw(e)` and `close()` from the caller straight into `sub`.
- It evaluates to `sub`'s **return value**, taken from `StopIteration.value`.

That makes it the delegation mechanism behind coroutines. Before 3.5, asyncio coroutines were literally generators using `yield from future`. `await x` is essentially `yield from x.__await__()`, restricted to awaitables. A `Task` drives the outermost coroutine with `.send(None)`, and a `Future` deep in the stack *yields itself* all the way up. **Day 5 builds on exactly this.**

### 1.4 Laziness gives you backpressure

A generator pipeline `read → parse → filter → batch` processes **one item at a time, pulled by the consumer**. Peak memory is O(batch size) instead of O(file size), and the producer does no work the consumer doesn't ask for. A generator object plus its frame costs a few hundred bytes. A list of 10M parsed records costs gigabytes.

---

## 2. Production Gotcha: cleanup inside a generator is deferred until `close()`

```python
def rows(conn) -> Iterator[Row]:
    cursor = conn.cursor()            # takes a server-side cursor / pooled connection
    try:
        yield from cursor
    finally:
        cursor.close()                # 🚨 when does this actually run?

for row in rows(conn):
    if row.id == target:
        break                         # consumer leaves early
```

When the consumer `break`s, the generator is **suspended, not finished**. `finally` runs only when someone calls `close()`. On CPython that usually happens as soon as the last reference disappears, which hides the bug in tests. It shows up when:

- the generator is **referenced elsewhere**: stored on `self`, held in a list, or captured by a traceback that is logged or stored;
- it's part of a **reference cycle**, so it waits for the cyclic GC (Day 1);
- you run on **PyPy** or another non-refcounting runtime;
- it's an **async generator**. These are *never* finalized synchronously. asyncio *schedules* `aclose()` as a new task via GC hooks, so cleanup runs at some later loop iteration. If the loop is already shutting down, it may never run at all, and the `finally` block (with its `await conn.release()`) is silently skipped.

Symptoms: `QueuePool limit of size N overflow reached`, `Too many open files`, locks held for no visible reason.

**Fixes:**

1. **Make closing explicit at the consumer:** `with contextlib.closing(gen):` for generators and `async with contextlib.aclosing(agen):` (3.10+) for async generators.
2. **Keep resource ownership outside the generator:** `with open(path) as f: for rec in parse(f): …`. The generator then only *transforms* the data, and the `with` statement owns the lifetime.
3. **Never `yield` inside a lock** (`with lock: yield x`). The lock is held for as long as the *consumer* takes between pulls.

**The second gotcha, single-pass exhaustion:** generators can be consumed only once. `log.info("n=%d", sum(1 for _ in events))` followed by `for e in events:` processes **zero** events, with no error. Also, `if events:` is always true for a generator. Type the parameter as `Iterable[T]` if one pass is enough. If you need several passes, accept `Sequence[T]` or `Collection[T]` and materialize explicitly.

---

## 3. Runnable Code Example

Save as `day03_generators.py`. It shows the state machine, `send`/`return`/`yield from`, a streaming pipeline with guaranteed cleanup, and both gotchas.

```python
"""
Day 3 — Generators as state machines: send/return, delegation, guaranteed cleanup.

Run:  python day03_generators.py        (Python 3.12+ for `def f[T]` syntax)
"""
from __future__ import annotations

import inspect
import io
import json
from collections.abc import Generator, Iterable, Iterator
from contextlib import closing
from dataclasses import dataclass
from itertools import islice
from typing import TextIO


# ---------------------------------------------------------------------------
# 1. A generator IS a state machine
# ---------------------------------------------------------------------------

def latency_tracker() -> Generator[float, float | None, dict[str, float]]:
    """send() latencies in, receive the running average out.
    Send None to finish; the summary is delivered via StopIteration.value."""
    count, total, peak = 0, 0.0, 0.0
    while True:
        sample = yield (total / count if count else 0.0)
        if sample is None:
            return {"count": count, "avg_ms": total / count if count else 0.0, "max_ms": peak}
        count += 1
        total += sample
        peak = max(peak, sample)


def tracker_with_report(sink: list[dict[str, float]]) -> Generator[float, float | None, None]:
    """`yield from` forwards next/send/throw/close AND captures the sub-generator's return value."""
    summary = yield from latency_tracker()
    sink.append(summary)


def demo_state_machine() -> None:
    gen = latency_tracker()
    print("state:", inspect.getgeneratorstate(gen))   # GEN_CREATED: no code has run
    next(gen)                                         # prime: run to the first `yield`
    print("state:", inspect.getgeneratorstate(gen))   # GEN_SUSPENDED
    for ms in (12.0, 48.5, 7.1):
        print(f"  sent {ms:>5} -> running avg {gen.send(ms):.2f}")
    try:
        gen.send(None)
    except StopIteration as stop:
        print("summary:", stop.value)
    print("state:", inspect.getgeneratorstate(gen))   # GEN_CLOSED

    sink: list[dict[str, float]] = []
    delegating = tracker_with_report(sink)
    next(delegating)
    delegating.send(100.0)            # goes straight through to latency_tracker
    delegating.send(300.0)
    try:
        delegating.send(None)         # inner returns -> outer resumes, appends, finishes
    except StopIteration:
        pass
    print("delegated summary:", sink[0])


# ---------------------------------------------------------------------------
# 2. Streaming pipeline with guaranteed cleanup
# ---------------------------------------------------------------------------

@dataclass(frozen=True, slots=True)
class LogEvent:
    level: str
    path: str
    latency_ms: float


def read_lines(stream: TextIO) -> Iterator[str]:
    """Owns the stream. `finally` runs on exhaustion, on exception, or on close()."""
    try:
        for line in stream:
            yield line.rstrip("\n")
    finally:
        stream.close()
        print("  [read_lines] finally ran -> stream closed")


def parse(lines: Iterable[str]) -> Iterator[LogEvent]:
    for line in lines:
        try:
            raw = json.loads(line)
            yield LogEvent(raw["level"], raw["path"], float(raw["latency_ms"]))
        except (json.JSONDecodeError, KeyError, TypeError, ValueError):
            continue  # production: increment a metric and sample the line to a dead-letter sink


def batched[T](items: Iterable[T], size: int) -> Iterator[list[T]]:
    """Like itertools.batched (3.12+), but yields lists. Peak memory = one batch."""
    it = iter(items)
    while batch := list(islice(it, size)):
        yield batch


def make_stream(n: int = 10) -> io.StringIO:
    rows = [
        json.dumps({"level": "INFO", "path": f"/v1/embed/{i}", "latency_ms": 40 + i * 70})
        for i in range(n)
    ]
    rows.insert(3, "not-json garbage line")
    return io.StringIO("\n".join(rows))


def first_slow_request(stream: TextIO, threshold_ms: float) -> LogEvent | None:
    lines = read_lines(stream)
    with closing(lines):  # the early `return` would otherwise leave `lines` suspended
        for event in parse(lines):
            if event.latency_ms > threshold_ms:
                return event
    return None


def demo_pipeline() -> None:
    print("\n-- early exit, WITH closing():")
    print("  found:", first_slow_request(make_stream(), threshold_ms=300))

    print("\n-- GOTCHA 1: early exit while a reference is still alive:")
    stream = make_stream()
    lines = read_lines(stream)
    for _ in lines:
        break                                     # consumer bails out early
    print("  stream.closed =", stream.closed)     # False: suspended at `yield`, finally not run
    lines.close()                                 # GeneratorExit at the paused yield -> finally
    print("  stream.closed =", stream.closed)     # True

    print("\n-- GOTCHA 2: generators are single-pass:")
    events = parse(read_lines(make_stream()))
    total = sum(1 for _ in events)
    print(f"  counted {total} events; second pass sees {len(list(events))}")

    print("\n-- batching for bulk writes (constant memory):")
    for batch in batched(parse(read_lines(make_stream())), size=4):
        print("  write batch:", [e.path for e in batch])


if __name__ == "__main__":
    demo_state_machine()
    demo_pipeline()
```

**Expected output (abridged):**

```text
state: GEN_CREATED
state: GEN_SUSPENDED
  sent  12.0 -> running avg 12.00
  sent  48.5 -> running avg 30.25
  sent   7.1 -> running avg 22.53
summary: {'count': 3, 'avg_ms': 22.53..., 'max_ms': 48.5}
state: GEN_CLOSED
delegated summary: {'count': 2, 'avg_ms': 200.0, 'max_ms': 300.0}

-- early exit, WITH closing():
  [read_lines] finally ran -> stream closed
  found: LogEvent(level='INFO', path='/v1/embed/4', latency_ms=320.0)

-- GOTCHA 1: early exit while a reference is still alive:
  stream.closed = False
  [read_lines] finally ran -> stream closed
  stream.closed = True
...
```

**Exercises (10 min):**

1. In `read_lines`, catch `GeneratorExit` and `yield "oops"` inside the handler. Call `.close()` and read the `RuntimeError`.
2. In `demo_state_machine`, call `gen.send(5.0)` *before* priming it. Read the `TypeError`.
3. Rewrite `latency_tracker` as a class with `__iter__`/`send`. Count how much state you have to manage by hand compared with what the frame stored for free.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `anthonywritescode generators yield from send` | **anthonywritescode** | Short, precise, CPython-maintainer-level detail |
| Deep dive | `David Beazley Generators The Final Frontier PyCon 2014` | **PyCon US** (David Beazley) | Generators → coroutines → event loops, built up from first principles. Watch the first half today and the rest after Day 5 |
| Practical | `mCoding python generators` | **mCoding** | Gotchas around exhaustion, `send`, and generator expressions |

---

[← Day 2 — Hashing](day-02-hashing.md) · [Day 4 — Two Pointers →](day-04-two-pointers.md)
