# Day 26 — Profiling: Measure Before You Optimize

[← Week 4](README.md) · **Track:** Python Systems · **Time box:** 55 min (concept 20 · gotcha 10 · code 20 · review 5)

> **Why this matters for platform work:** "The service is slow" can mean a CPU hotspot, a blocked event loop (Day 5), lock contention, GC pauses (Day 1), or waiting on a dependency. Each one needs a different tool, and the wrong tool gives you a confident, wrong answer. Profiling skills are what separate "I rewrote it in Rust" from "I changed one line".

---

## 1. Core Concept

### 1.1 Two families of profilers

| | **Deterministic (tracing)** | **Sampling (statistical)** |
|---|---|---|
| Examples | `cProfile`, `profile`, `line_profiler` | `py-spy`, `austin`, `scalene`, Linux `perf` (with `-X perf`, 3.12+) |
| Mechanism | A hook fires on **every** function call and return (`sys.setprofile`, or `sys.monitoring` events in 3.12+) | Interrupts or inspects the process at a fixed rate (e.g. 100 Hz) and records the stack |
| Output | Exact call counts, per-function time | Statistical distribution of where time goes |
| Overhead | High and **non-uniform**: roughly 1.3–3× slower, and much worse for code making many tiny calls | Low (a few %), and constant |
| Attach to a running prod process? | No, you have to wrap the code | **Yes**: `py-spy` reads process memory from outside, with no code change and no restart |
| Best for | "How many times is X called?", reproducible local investigation | "Where is prod spending its time *right now*?" |

**The distortion problem:** `cProfile` adds a fixed cost to *every call*. Code made of millions of small function calls looks much slower than it is, while code that spends its time in a few big C calls (or inline bytecode such as `x in some_list`) looks cheap. **cProfile is function-granular.** The time spent on a `list.__contains__` scan inside your function is reported as that function's own time (`tottime`), with no separate entry, which is exactly what you'll see in today's example.

### 1.2 Reading `pstats`

| Column | Meaning |
|---|---|
| `ncalls` | Number of calls (`3/1` = 3 total, 1 primitive, i.e. non-recursive) |
| `tottime` | Time spent in the function **itself**, excluding sub-calls. **Sort by this to find hotspots** |
| `cumtime` | Time including everything it called. **Sort by this to find which high-level path is slow** |
| `percall` | `tottime/ncalls` and `cumtime/ncalls` |

Workflow: sort by `cumtime` to find the expensive *subsystem*, then by `tottime` within it to find the actual *hot function*.

### 1.3 Sampling in production with py-spy

```bash
py-spy top --pid 12345                          # live "top" for Python functions
py-spy dump --pid 12345                         # stack of every thread right now (stuck worker?)
py-spy record -o profile.svg --pid 12345 -d 30  # 30 s flame graph
py-spy record --idle --gil ...                  # include idle threads / only GIL-holding samples
```

In containers you need `SYS_PTRACE` capability (or run py-spy as a sidecar sharing the PID namespace). `py-spy dump` is the fastest way to answer "what is this hung worker doing?". In asyncio services it shows exactly which sync call is blocking the loop.

### 1.4 Reading a flame graph

- **x-axis is not time.** It's the share of samples, and frames are sorted **alphabetically** to merge identical stacks. The width of a box = the fraction of time that function was on the stack.
- **y-axis is stack depth.** Callers are below, callees above. (Icicle graphs flip this.)
- **Look for wide plateaus**: wide boxes with nothing (or something thin) above them. That's time spent *in* that function, i.e. its `tottime`. Wide boxes with wide children are just passing time through to their callees.
- A wide `select`/`epoll_wait` at the top in an asyncio service means *idle*, not slow. Use `--idle` deliberately, and don't optimize waiting.

### 1.5 Memory: memray and tracemalloc

`tracemalloc` (Day 1) is built in and good for snapshot diffs. **memray** (Bloomberg) tracks every allocation, including native ones from C extensions with `--native`. `memray run -o out.bin app.py` followed by `memray flamegraph out.bin` shows *who allocated the peak*, and `memray run --live` gives a TUI. Use it when RSS grows but the Python heap looks flat, or when you need to see the peak during a batch job.

### 1.6 Benchmarking without lying to yourself

- **Clocks:** `time.perf_counter()` is monotonic and has the highest available resolution, so use it for durations. `time.time()` is wall-clock time: it can jump with NTP adjustments and historically had about 16 ms resolution on Windows. Never use it for durations. `time.process_time()` counts CPU time only, so it excludes sleep and I/O wait. A big gap between wall time and process time means you're *waiting*, not computing.
- **`timeit`** runs the statement many times, **disables GC during timing by default**, and uses `perf_counter`. Report the **minimum** of several repeats: noise only ever adds time, so the minimum is the best estimate of the code's intrinsic cost. (`pyperf` instead reports mean ± standard deviation across processes.)
- **Pitfalls:** measuring the first call (cold caches, lazy imports); a dataset too small, or one that fits in L2 cache unlike prod; constant-folded expressions (`timeit("2**10")` measures nothing); CPU frequency scaling; noisy neighbours on shared CI runners.
- **`pyperf`** (and `pyperformance`) spawns multiple processes, calibrates loop counts, does warmups and reports statistics, and `python -m pyperf system tune` reduces OS noise. Use it before you claim "X% faster" in a PR.

---

## 2. Production Gotcha: profiling the wrong thing

### 2a. cProfile on an async service says "everything is in `select`"

You wrap an asyncio app in `cProfile` and the top entry by `cumtime` is `run_forever` / `select`, with your handlers barely visible. Coroutines suspend and resume many times, so per-call attribution is muddled, and idle time dominates. **What to do instead:** run a sampling profiler on the live process (`py-spy record --pid`), and add a loop-lag monitor (Day 5) and per-request timing spans (Week 6, OpenTelemetry). If the problem is *waiting* (slow DB, upstream latency), a CPU profiler will never find it. That's a tracing problem, not a profiling problem.

### 2b. Optimizing from a micro-benchmark

Someone shows `timeit` proving that `orjson` is 6× faster than `json`, and swaps it in. p99 doesn't change, because serialization was 2% of request time. **Amdahl's law:** making something that accounts for fraction *p* of the time *s* times faster gives an overall speedup of `1 / ((1 − p) + p/s)`. At p = 0.02, even s = ∞ gives 1.02×. **Always profile the real workload first, then benchmark the hotspot.**

### 2c. Measuring with `time.time()` in a tight loop

On some platforms `time.time()` only advances every ~16 ms, so it reports `0.0` and then `0.0156` for operations that take 1 ms. NTP slews can also make durations negative. Use `perf_counter()` (or `perf_counter_ns()`) for durations and `time.time()` only for timestamps.

---

## 3. Runnable Code Example

Save as `day26_profiling.py`. It profiles a realistic slow report function with `cProfile`, reads the stats programmatically, fixes the two hotspots, and verifies the speedup with `timeit`, using the minimum of several repeats.

```python
"""
Day 26 - Profile, find the hotspot, fix it, and prove the speedup properly.

Run:  python day26_profiling.py        (Python 3.12+)
"""
from __future__ import annotations

import cProfile
import pstats
import random
import re
import time
import timeit
from collections.abc import Callable
from dataclasses import dataclass
from functools import lru_cache


@dataclass(frozen=True, slots=True)
class Event:
    request_id: str
    path: str
    latency_ms: float


def make_events(n: int, seed: int = 7) -> list[Event]:
    rng = random.Random(seed)
    return [
        Event(
            request_id=f"req-{rng.randrange(n // 2)}",                 # retries -> duplicate IDs
            path=f"/V1/users/{rng.randrange(1000)}/orders/{rng.randrange(50)}",
            latency_ms=rng.expovariate(1 / 40),
        )
        for _ in range(n)
    ]


def p99(values: list[float]) -> float:
    ordered = sorted(values)
    return ordered[min(len(ordered) - 1, int(0.99 * len(ordered)))]


# ---------------------------------------------------------------------------
# Slow version (the kind of code that ships)
# ---------------------------------------------------------------------------
def normalize_slow(path: str) -> str:
    return re.sub(r"/\d+", "/:id", path.lower())         # pattern looked up in re's cache each call


def slow_report(events: list[Event]) -> dict[str, float]:
    seen: list[str] = []
    per_route: dict[str, list[float]] = {}
    for e in events:
        if e.request_id in seen:                          # O(n) scan -> O(n^2) overall
            continue
        seen.append(e.request_id)
        per_route.setdefault(normalize_slow(e.path), []).append(e.latency_ms)
    return {route: p99(v) for route, v in per_route.items()}


# ---------------------------------------------------------------------------
# Fast version (after reading the profile)
# ---------------------------------------------------------------------------
_NUMERIC_SEGMENT = re.compile(r"/\d+")


@lru_cache(maxsize=65_536)                                # module-level pure function: safe (Day 1)
def normalize_fast(path: str) -> str:
    return _NUMERIC_SEGMENT.sub("/:id", path.lower())


def fast_report(events: list[Event]) -> dict[str, float]:
    seen: set[str] = set()                                # O(1) membership (Day 2)
    per_route: dict[str, list[float]] = {}
    for e in events:
        if e.request_id in seen:
            continue
        seen.add(e.request_id)
        per_route.setdefault(normalize_fast(e.path), []).append(e.latency_ms)
    return {route: p99(v) for route, v in per_route.items()}


# ---------------------------------------------------------------------------
# Profiling helpers
# ---------------------------------------------------------------------------
def top_functions(profiler: cProfile.Profile, n: int = 5) -> None:
    """Compact view of pstats: sort by tottime (self time) to find the real hotspot."""
    stats = pstats.Stats(profiler)
    rows = []
    for (filename, lineno, func), (cc, ncalls, tottime, cumtime, _callers) in stats.stats.items():  # type: ignore[attr-defined]
        rows.append((tottime, cumtime, ncalls, func))
    rows.sort(reverse=True)
    print(f"  {'function':<22} {'ncalls':>8} {'tottime':>9} {'cumtime':>9}")
    for tottime, cumtime, ncalls, func in rows[:n]:
        print(f"  {func[:22]:<22} {ncalls:>8} {tottime:>8.3f}s {cumtime:>8.3f}s")


def best_of(fn: Callable[[list[Event]], object], events: list[Event], repeat: int = 3) -> float:
    # setup runs before EVERY repeat: start each run with a cold cache, or repeats 2..n
    # would measure a warm lru_cache and flatter the fast version (a classic pitfall).
    timer = timeit.Timer(lambda: fn(events), setup=normalize_fast.cache_clear)
    return min(timer.repeat(repeat=repeat, number=1))     # min = least-noise estimate


def main() -> None:
    for name in ("time", "perf_counter"):
        info = time.get_clock_info(name)
        print(f"clock {name:<12} monotonic={info.monotonic!s:<5} resolution={info.resolution:.1e}s")

    events = make_events(10_000)

    print("\n== cProfile: slow_report, sorted by tottime")
    with cProfile.Profile() as profiler:
        slow_result = slow_report(events)
    top_functions(profiler)

    print("\n== cProfile: fast_report, sorted by tottime")
    normalize_fast.cache_clear()
    with cProfile.Profile() as profiler:
        fast_result = fast_report(events)
    top_functions(profiler)

    assert slow_result == fast_result, "optimization changed behaviour!"

    print("\n== timeit (min of 3 runs, no profiler overhead)")
    slow_t = best_of(slow_report, events)
    fast_t = best_of(fast_report, events)
    print(f"  slow_report: {slow_t * 1000:8.1f} ms")
    print(f"  fast_report: {fast_t * 1000:8.1f} ms   ({slow_t / fast_t:.0f}x faster)")


if __name__ == "__main__":
    main()
```

**Expected output (abridged; timings vary by machine):**

```text
clock time         monotonic=False resolution=1.6e-02s
clock perf_counter monotonic=True  resolution=1.0e-07s

== cProfile: slow_report, sorted by tottime
  function                 ncalls   tottime   cumtime
  slow_report                   1    0.577s    0.628s
  <method 'sub' of 're.P     4319    0.016s    0.016s
  _compile                   4319    0.007s    0.011s
  sub                        4319    0.006s    0.034s
  normalize_slow             4319    0.006s    0.042s

== cProfile: fast_report, sorted by tottime
  function                 ncalls   tottime   cumtime
  fast_report                   1    0.016s    0.039s
  <method 'sub' of 're.P     4144    0.011s    0.011s
  normalize_fast             4144    0.005s    0.017s
  <method 'setdefault' o     4319    0.002s    0.002s
  <method 'lower' of 'st     4144    0.002s    0.002s

== timeit (min of 3 runs, no profiler overhead)
  slow_report:    566.6 ms
  fast_report:     15.5 ms   (37x faster)
```

**How to read it:**

- In the slow profile, `slow_report` has the largest **`tottime`**, even though it makes no expensive *calls*. The `in seen` scan of a list is inline bytecode, so cProfile charges it to the enclosing function. Without knowing that, you might "optimize" `re.sub` first, which is the second-largest entry and nowhere near the main cost.
- Both fixes come from earlier days: `list` → `set` (Day 2, hashing) and a module-level `lru_cache` on a pure function (Day 1).
- The first line shows `time.time()` resolution of **16 ms** on this Windows machine (Python 3.12), versus 100 ns for `perf_counter`.
- A draft of this script reported **126×** instead of 37×. Without the `setup=normalize_fast.cache_clear`, repeats 2 and 3 measured a *warm* `lru_cache`, and `min()` picked the flattering run. Production traffic has a much colder cache than repeat #3 of a benchmark. Make sure every repeat measures the same thing.
- The `assert slow_result == fast_result` matters as much as the timing. **An optimization without an equivalence check is a refactor with unknown behaviour.**

**Exercises (10 min):**

1. Change `make_events(10_000)` to `20_000`. The slow version roughly quadruples while the fast version roughly doubles. That's what $O(n^2)$ looks like in a profile.
2. `pip install py-spy`, add `time.sleep(30)` plus a loop calling `slow_report` at the end, and run `py-spy top --pid <pid>` from another terminal. Then try `py-spy record -o out.svg` and open the flame graph.
3. Replace `best_of`'s `min` with a mean, run it 5 times, and compare how much each statistic varies.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `mCoding profiling python cProfile` | **mCoding** | Concise cProfile + snakeviz walkthrough, plus "measure first" discipline |
| Production | `py-spy sampling profiler python production` | **PyCon / EuroPython** talks | Profiling live services safely, flame-graph interpretation |
| Memory | `memray Python memory profiler Pablo Galindo` | **PyCon / EuroPython** (Pablo Galindo Salgado) | Native-aware allocation tracking from the memray authors |
| Benchmarking | `Victor Stinner pyperf benchmark` | **PyCon / FOSDEM** recordings | Why naive benchmarks lie and how pyperf fixes it |

**Reading:** Brendan Gregg's flame graph pages (the original description of how to read them).

---

[← Day 25 — Intervals](day-25-intervals.md) · [Day 27 — Grid BFS/DFS →](day-27-graph-grid-traversal.md)
