# Day 12 — The GIL, Threads vs Processes vs asyncio, and fork vs spawn

[← Week 2](README.md) · **Track:** Python Systems · **Time box:** 60 min (concept 20 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** Choosing between a thread pool, a process pool and an event loop is an everyday decision when serving models, preprocessing data or fanning out API calls. A wrong guess costs 4× in throughput, or produces worker processes that hang forever on their first log line.

**Prerequisites:** Day 1 (refcounts) and Day 5 (`to_thread`, the event loop).

---

## 1. Core Concept

### 1.1 What the GIL protects

The **Global Interpreter Lock** is a mutex per interpreter. A thread must hold it to execute Python bytecode or touch Python objects. It exists because CPython's internals aren't thread-safe on their own: reference counts (Day 1) are plain non-atomic integers, and the allocator, type caches and many built-in containers assume exclusive access. A single lock makes all of that safe, keeps single-threaded code fast, and makes C extensions easy to write.

The trade-off: **only one thread runs Python bytecode at a time** per process, however many cores you have.

### 1.2 How threads take turns (the "new GIL", 3.2+)

```text
Thread B wants the GIL, A holds it:
  B waits on a condition variable with a timeout = sys.getswitchinterval()  (default 0.005 s)
  timeout expires -> B sets a "drop request" flag in A's eval breaker
  A checks the eval breaker at safe points (function entry, backward jumps in loops...)
  A releases the GIL, signals; B acquires it; A now waits in turn
```

Consequences:

- A CPU-bound thread is preempted roughly every **5 ms**. Not after N bytecodes (that was the pre-3.2 behaviour).
- **Convoy effect:** an I/O thread that wakes up (a socket became readable) may have to wait up to a full switch interval to get the GIL back from a CPU-hogging thread. Mixing CPU-heavy work with latency-sensitive I/O threads in the same process inflates tail latency.

### 1.3 When the GIL is released, and threads do run in parallel

- **Blocking system calls:** `socket.recv`, `file.read`, `time.sleep`, `select`, `subprocess.wait`, DNS lookups. CPython wraps them in `Py_BEGIN_ALLOW_THREADS … Py_END_ALLOW_THREADS`.
- **C extensions doing long native work:** NumPy ufuncs and BLAS on large arrays, `hashlib` on buffers larger than about 2 KiB, `zlib`/`lzma`, PyTorch ops, Rust-based tokenizers, `orjson` in some paths.

So threads give you **real parallelism for I/O and for native code that releases the GIL**. They give none for pure-Python loops.

### 1.4 The GIL doesn't make your code thread-safe

`self.count += 1` is several bytecodes (load, add, store). A switch can happen between the load and the store, so two threads both read 41 and both write 42. Some single operations happen to be atomic in CPython (`list.append`, `dict[k] = v`, `deque.popleft`), but that's an implementation detail, and it isn't true of compound operations (`if k not in d: d[k] = …`). **Protect shared mutable state with `threading.Lock`**, or avoid sharing it by passing messages through `queue.Queue`.

### 1.5 Processes: start methods and pickling

Each process has its own interpreter and its own GIL, so CPU-bound work scales across cores. The costs are start-up time, memory, and **every argument, result and exception being pickled** across a pipe.

| Start method | How | Default on | Notes |
|---|---|---|---|
| `fork` | `fork()` the parent: a copy-on-write clone | Linux up to 3.13 | Fast; inherits everything, *including the state of locks held by other threads*. **Unsafe once the parent has threads** (§2) |
| `spawn` | A fresh interpreter **re-imports your main module**, then unpickles the target | Windows, macOS (3.8+) | Slow start (hundreds of ms); requires the `if __name__ == "__main__":` guard; only picklable targets |
| `forkserver` | A clean single-threaded server process forks workers on request | Linux from 3.14 | Fork-like speed without inheriting the parent's threads |

**Pickling costs are easy to underestimate.** `pool.map(score, [(big_model, x) for x in rows])` pickles `big_model` *once per task*. Load heavy state **once per worker** with `initializer=`, pass IDs or paths instead of payloads, use `multiprocessing.shared_memory` for large arrays, and use `chunksize=` in `map` to amortize IPC overhead. Lambdas, closures, locks, sockets and open files can't be pickled at all.

`concurrent.futures` gives threads and processes the same interface (`submit`, `map`, `as_completed`). `ProcessPoolExecutor(max_tasks_per_child=N)` (3.11+) recycles workers, which helps with Day 1 fragmentation. A worker killed by the OOM killer surfaces as `BrokenProcessPool`, and the pool can't be used afterwards, so recreate it.

### 1.6 Decision table

| Workload | Use | Why |
|---|---|---|
| Thousands of concurrent network calls with async-capable libraries | **asyncio** | Lowest overhead per connection; one thread |
| Blocking I/O through sync-only libraries (boto3, legacy SDKs) | **Threads** (`ThreadPoolExecutor`, `asyncio.to_thread`) | The GIL is released during the I/O |
| CPU-heavy **native** work (NumPy, PyTorch, tokenizers, compression) | **Threads** are often enough | Native code releases the GIL; no pickling cost |
| CPU-heavy **pure-Python** work (parsing, feature engineering loops) | **Processes** | The only way to use more than one core with a GIL build |
| A mix (an async API doing CPU-heavy preprocessing) | asyncio + `ProcessPoolExecutor` via `loop.run_in_executor` | Keeps the event loop responsive |

### 1.7 Free-threaded CPython (PEP 703)

- **3.13t** shipped an *experimental* build with no GIL (`python3.13t`). **3.14t** became officially supported (PEP 779), with much lower single-threaded overhead, but it's still a *separate build*. The default `python` still has a GIL.
- It uses biased reference counting, per-object locks inside built-in containers, and `mimalloc` (Day 1).
- A C extension that doesn't declare free-threading support **re-enables the GIL at import time** (with a warning). Check at runtime with `sys._is_gil_enabled()`.
- Races your code always had become **more likely to show up**. The locking discipline in §1.4 matters more, not less.
- 3.14 also adds `concurrent.interpreters` (PEP 734) and an `InterpreterPoolExecutor`: multiple interpreters in one process, each with its own GIL.

---

## 2. Production Gotcha: `fork()` after threads exist → children that hang forever

```python
# app startup
start_metrics_pusher_thread()        # or: Sentry SDK, OpenTelemetry BatchSpanProcessor, a gRPC client...
logging.basicConfig(...)

# later, inside a request or batch job
with multiprocessing.Pool(8) as pool:          # 🚨 Linux <= 3.13 default start method: fork
    results = pool.map(transform, chunks)      #    -> some workers hang; pool.map never returns
```

`fork()` copies the entire memory image, **including the state of every lock**, but only the **calling thread** keeps running in the child. If any other thread held a lock at the instant of the fork (the `logging` handler lock while writing a record, glibc's malloc arena lock, an OpenBLAS thread-pool mutex, gRPC's internal locks), that lock stays **held forever** in the child, because the thread that would release it doesn't exist there. The child deadlocks the first time it logs, allocates in that arena, or makes a gRPC call.

It's intermittent (it depends on timing), it doesn't show up on macOS/Windows dev machines (which default to `spawn`), and it shows up as "batch job stuck at 87%" with workers idling at 0% CPU.

**Detect:** Python 3.12+ emits `DeprecationWarning: This process ... is multi-threaded, use of fork() may lead to deadlocks in the child.` Turn warnings on in CI (`-W error::DeprecationWarning`). On a stuck worker, `py-spy dump --pid <child>` shows it blocked in `acquire`.

**Fix:**

1. Choose the start method explicitly: `ctx = multiprocessing.get_context("forkserver")` (or `"spawn"`), then `ctx.Pool(...)`, or `ProcessPoolExecutor(mp_context=ctx)`. Don't rely on the platform default, which also changed in 3.14.
2. If you must fork (prefork servers like gunicorn), start background threads **after** the fork, for example in a `post_fork` hook, and never in module import side effects.
3. Libraries can protect themselves with `os.register_at_fork(before=..., after_in_child=...)`.

**The second gotcha, spawn without a `__main__` guard:** with `spawn`, every worker re-imports your main module. Without `if __name__ == "__main__":`, the pool-creating code runs again in each child, and you get `RuntimeError: An attempt has been made to start a new process before the current process has finished its bootstrapping phase`. Notebooks and `python -c` have their own `__main__` quirks, so run pool code from a real script or module.

---

## 3. Runnable Code Example

Save as `day12_gil.py` and run it as a script (spawn needs the `__main__` guard). It measures CPU-bound and I/O-bound work under threads and processes, shows a real race condition, and compares per-task pickling with a worker initializer. It's stdlib only and runs on Windows, macOS and Linux.

```python
"""
Day 12 — The GIL in practice: CPU vs I/O under threads and processes,
a real race condition, and the worker-initializer pattern.

Run:  python day12_gil.py        (Python 3.12+; run as a script, not in a notebook)
"""
from __future__ import annotations

import multiprocessing as mp
import pickle
import sys
import threading
import time
from collections.abc import Callable
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor
from typing import Any

WORKERS = 4


def timed(label: str, fn: Callable[[], Any]) -> float:
    start = time.perf_counter()
    fn()
    elapsed = time.perf_counter() - start
    print(f"  {label:<26} {elapsed:6.2f}s")
    return elapsed


# ---------------------------------------------------------------------------
# 1. CPU-bound vs I/O-bound
# ---------------------------------------------------------------------------
def cpu_task(n: int) -> int:
    """Pure-Python arithmetic: holds the GIL the entire time."""
    total = 0
    for i in range(n):
        total += i * i % 7
    return total


def io_task(seconds: float) -> float:
    time.sleep(seconds)  # blocking syscall: GIL released while waiting
    return seconds


def demo_cpu_vs_io(ctx: Any) -> None:
    n = 2_000_000
    jobs = [n] * WORKERS
    print(f"-- CPU-bound: {WORKERS} x cpu_task({n:,})")
    timed("sequential", lambda: [cpu_task(j) for j in jobs])
    with ThreadPoolExecutor(WORKERS) as threads:
        timed("threads (GIL-bound)", lambda: list(threads.map(cpu_task, jobs)))
    with ProcessPoolExecutor(WORKERS, mp_context=ctx) as procs:
        list(procs.map(cpu_task, [1] * WORKERS))           # warm-up: pay process start-up first
        timed("processes", lambda: list(procs.map(cpu_task, jobs)))

    print("-- I/O-bound: 20 x sleep(0.1)")
    timed("sequential", lambda: [io_task(0.1) for _ in range(20)])
    with ThreadPoolExecutor(20) as threads:
        timed("threads (GIL released)", lambda: list(threads.map(io_task, [0.1] * 20)))


# ---------------------------------------------------------------------------
# 2. The GIL does not make read-modify-write atomic
# ---------------------------------------------------------------------------
class Counter:
    def __init__(self) -> None:
        self.value = 0
        self._lock = threading.Lock()

    def unsafe_increment(self) -> None:
        current = self.value
        time.sleep(0)            # stands in for ANY GIL release point: a log call, I/O, a switch
        self.value = current + 1

    def safe_increment(self) -> None:
        with self._lock:
            current = self.value
            time.sleep(0)
            self.value = current + 1


def demo_race() -> None:
    print("-- race condition: 8 threads x 500 increments (expected 4000)")
    for method in ("unsafe_increment", "safe_increment"):
        counter = Counter()

        def work() -> None:
            for _ in range(500):
                getattr(counter, method)()

        threads = [threading.Thread(target=work) for _ in range(8)]
        for t in threads:
            t.start()
        for t in threads:
            t.join()
        print(f"  {method:<17} -> {counter.value}")


# ---------------------------------------------------------------------------
# 3. Pickling cost: ship state per task vs load once per worker
# ---------------------------------------------------------------------------
_MODEL: dict[str, float] = {}


def build_model(size: int = 200_000) -> dict[str, float]:
    return {f"feature_{i}": i * 0.5 for i in range(size)}


def init_worker() -> None:
    global _MODEL
    _MODEL = build_model()                       # runs once in each worker process


def score_with_global(key: str) -> float:
    return _MODEL.get(key, 0.0)


def score_with_payload(args: tuple[dict[str, float], str]) -> float:
    model, key = args                            # the whole model was pickled for THIS task
    return model.get(key, 0.0)


def demo_pickling(ctx: Any) -> None:
    model = build_model()
    keys = [f"feature_{i}" for i in range(0, 200_000, 5_000)]   # 40 tasks
    size_mb = len(pickle.dumps(model)) / 1e6
    print(f"-- pickling: 40 tasks, model pickles to {size_mb:.1f} MB")
    with ProcessPoolExecutor(WORKERS, mp_context=ctx) as procs:
        list(procs.map(cpu_task, [1] * WORKERS))
        timed("model sent per task", lambda: list(procs.map(score_with_payload, [(model, k) for k in keys])))
    with ProcessPoolExecutor(WORKERS, mp_context=ctx, initializer=init_worker) as procs:
        list(procs.map(cpu_task, [1] * WORKERS))
        timed("initializer (once/worker)", lambda: list(procs.map(score_with_global, keys)))


if __name__ == "__main__":
    gil = getattr(sys, "_is_gil_enabled", lambda: True)()
    print(f"python {sys.version.split()[0]} | GIL enabled: {gil} | "
          f"switch interval: {sys.getswitchinterval()}s\n")
    ctx = mp.get_context("spawn")     # explicit and portable; never rely on the platform default
    demo_cpu_vs_io(ctx)
    demo_race()
    demo_pickling(ctx)
```

**Expected output (timings vary by machine; a 12-core Windows laptop shown; the whole script takes about 30 s, mostly the per-task pickling demo):**

```text
python 3.12.6 | GIL enabled: True | switch interval: 0.005s

-- CPU-bound: 4 x cpu_task(2,000,000)
  sequential                   1.74s
  threads (GIL-bound)          1.88s
  processes                    0.65s
-- I/O-bound: 20 x sleep(0.1)
  sequential                   2.01s
  threads (GIL released)       0.11s
-- race condition: 8 threads x 500 increments (expected 4000)
  unsafe_increment  -> 646
  safe_increment    -> 4000
-- pickling: 40 tasks, model pickles to 5.1 MB
  model sent per task          5.34s
  initializer (once/worker)    0.02s
```

How to read it:

- **CPU-bound with threads** is no faster than sequential, and often slightly slower because of GIL hand-off overhead. Processes scale with cores.
- **I/O-bound with threads** gets nearly the full 20× speed-up, because `sleep` releases the GIL.
- The unsafe counter loses most of its updates, *with the GIL enabled*.
- Shipping the model with every task pickles it 40 times. The initializer pays the cost once per worker. (Including worker start-up in the second timing would narrow the gap; the gap widens with model size and task count.)

**Exercises (10 min):**

1. Replace `cpu_task`'s loop with `hashlib.sha256(b"x" * 50_000_000).hexdigest()` and re-run the threads benchmark. What changed, and why?
2. Delete the `if __name__ == "__main__":` guard (keep the body at module level) and run it. Read the `RuntimeError`.
3. Run `sys.setswitchinterval(0.0001)` before `demo_race` and compare the unsafe result. Then remove `time.sleep(0)` from `unsafe_increment` and run it a few times. Does the race disappear, or just get rarer?

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `David Beazley Understanding the Python GIL` | **PyCon / conference recordings** (David Beazley) | The classic dissection of GIL hand-off and the convoy effect. The mechanics still apply |
| Deep dive | `Larry Hastings Python GIL PyCon` / `Sam Gross nogil free threading` | **PyCon US** | Why removing the GIL is hard, and how PEP 703 does it |
| Practical | `Corey Schafer threading tutorial` / `Corey Schafer multiprocessing tutorial` | **Corey Schafer** | Clear `concurrent.futures` usage patterns |
| Current | `mCoding free threaded python` | **mCoding** | What the free-threaded build changes for your code |

---

[← Day 11 — Binary Search](day-11-binary-search.md) · [Day 13 — Fast/Slow Pointers →](day-13-linked-lists-fast-slow.md)
