# Day 1 — CPython Memory Model: Refcounting, Cyclic GC, and pymalloc

[← Week 1](README.md) · **Track:** Python Systems · **Time box:** 50 min (concept 15 · gotcha 10 · code 20 · review 5)

> **Why this matters for platform work:** Long-running services such as API workers, inference servers and Celery consumers die from memory problems far more often than from CPU problems. If you can't tell a leak from fragmentation, you'll either restart pods on a cron forever or spend days chasing a leak that doesn't exist.

---

## 1. Core Concept

### 1.1 Names are references; objects carry the count

Every CPython object begins with a `PyObject` header containing two fields: `ob_refcnt` (reference count) and `ob_type` (pointer to its type). Together they take 16 bytes on 64-bit builds. Variables, list slots, dict values, closure cells and frame locals all store only a `PyObject*`.

```python
a = [1, 2, 3]
b = a          # copies a pointer, increments ob_refcnt. Nothing is cloned.
```

So "assignment" in Python is always pointer binding. Copying happens only through explicit calls such as `copy.copy`, `list(x)` or slicing.

### 1.2 Reference counting: deterministic, immediate

Each new reference triggers `Py_INCREF` and each dropped reference triggers `Py_DECREF`. When the count reaches zero, the type's `tp_dealloc` runs **immediately, inline, on the thread that dropped the last reference**. Deallocating a container decrefs its children, which can cascade. For example, freeing a dict with 10M entries happens synchronously inside whichever `del` or reassignment dropped the last reference, and you'll see it as a latency spike in that request.

Things to know:

- `sys.getrefcount(x)` reports one more than you expect because passing `x` as an argument creates a temporary reference.
- **Immortal objects (3.12+, PEP 683):** `None`, `True`, `False`, small ints (−5..256) and many interned strings have a fixed sentinel refcount. `sys.getrefcount(None)` returns `4294967295` on 64-bit 3.12. Incref and decref on these objects are no-ops, so they never write to their memory page. This matters for `fork()`-based servers (next point).
- **Copy-on-write erosion:** after `fork()`, parent and child share memory pages until either one writes to them. *Reading* a Python object writes to it, because the refcount changes. Over time, gunicorn/uWSGI prefork workers "unshare" the parent's heap and memory grows by N × heap. Mitigations are `gc.freeze()` just before forking (it moves all current objects into a permanent generation that the GC never traverses) and loading large read-only data *after* fork, or into shared memory or mmap.

### 1.3 The cyclic garbage collector handles what refcounting can't

Refcounting alone can't free a cycle: `a.peer = b; b.peer = a; del a, b` leaves both objects with refcount 1, unreachable forever. CPython's supplemental **cycle collector** finds these.

Only **container objects**, meaning types that implement `tp_traverse` (lists, dicts, sets, class instances, closures, frames), are tracked. Ints, floats and strings can't hold references, so they're never tracked.

How a collection finds garbage:

1. For each tracked object in the set being collected, copy `ob_refcnt` into a scratch field `gc_refs`.
2. Call `tp_traverse` on each object. For every referent that is *also in the set*, decrement its `gc_refs`. This subtracts all internal references.
3. Any object with `gc_refs > 0` is referenced from *outside* the set (a stack frame, a global, a C extension). It's a root, and everything reachable from it is alive.
4. Everything else is unreachable. The collector handles weakref callbacks, runs finalizers (`__del__`; since 3.4 / PEP 442, cycles containing `__del__` are collectable), then calls `tp_clear` to break the references so refcounting frees the objects.

**Generations:** Most objects die young, so new objects go into the youngest generation, and survivors get promoted. A young collection triggers when *allocations − deallocations* of tracked objects exceeds a threshold (`gc.get_threshold()`, default `(700, 10, 10)` through 3.13). Full collections have an extra heuristic so they don't become quadratic on large heaps. **3.14** replaced this with an *incremental* collector with young/old generations, which cuts worst-case pause times. Everything above still applies.

A classic built-in cycle: an exception holds `__traceback__`, the traceback holds frames, and the frames hold locals, *including the exception itself*. That's why Python **implicitly deletes `e` at the end of `except E as e:`**. If you store the exception (`self.last_error = e`), you keep every local of every frame in its traceback alive.

### 1.4 The allocator: pymalloc, and why RSS doesn't go down

Requests of **≤ 512 bytes** (most Python objects) go to `pymalloc`. Larger requests go to the system `malloc`.

```text
arena (1 MiB, mmap'd)  →  pools (16 KiB each; one size class per pool)  →  blocks (16-byte size classes)
```

**An arena is returned to the OS only when every pool in it is empty.** Suppose you load 2M rows, process them and drop them. If a few long-lived small objects were allocated in the same arenas during the spike, those arenas stay pinned. Python *reuses* that memory, but your container's RSS stays at the peak. This is **fragmentation, not a leak**, and the two need different fixes.

| Symptom | `tracemalloc` (Python heap) | RSS | Diagnosis | Fix |
|---|---|---|---|---|
| Grows forever | Grows | Grows | **Leak**: something holds references | Find the referrer (snapshot diffs, `gc.get_referrers`, memray) |
| Jumps after spike, then flat | Back to baseline | Stays high | **Fragmentation / high-water mark** | Stream instead of materializing; recycle workers (`gunicorn --max-requests` + `--max-requests-jitter`, `celery --max-tasks-per-child`, `uvicorn --limit-max-requests`) |
| Grows, Python heap flat | Flat | Grows | **Native leak** (C extension, numpy, gRPC buffers) | `memray --native`, jemalloc profiling |

> **Free-threaded builds (3.13t / 3.14t)** use biased reference counting (the owning thread does cheap non-atomic increments) and `mimalloc` in place of pymalloc. The semantics above still hold, but the allocator diagnostics differ.

---

## 2. Production Gotcha: `@lru_cache` on a method is an unbounded instance leak

```python
class ModelClient:
    def __init__(self, endpoint: str) -> None:
        self.endpoint = endpoint
        self.tokenizer = load_tokenizer()     # ~200 MB

    @lru_cache(maxsize=1024)                  # 🚨
    def embed(self, text: str) -> list[float]: ...
```

**What goes wrong:** `lru_cache` wraps the *function* stored on the *class*. The cache is shared by every instance, and its keys include `self`. Every `ModelClient` that ever called `embed` stays referenced by the cache until it gets evicted, along with its 200 MB tokenizer. With `maxsize=None` this never happens. A service that builds a client per request or per tenant leaks steadily. `weakref`s to the instances never die, and `gc.collect()` doesn't help because nothing is a cycle: the cache genuinely owns them.

It also hurts the cache itself: hits are partitioned per instance, so two clients with identical config never share entries.

**Fixes, in order of preference:**

1. **Move the cache to a module-level pure function keyed on primitives:** `_embed(endpoint: str, text: str)`. Instances stay disposable, and the cache is shared across equivalent instances.
2. **Per-instance cache in `__init__`:** `self._cache: dict[str, list[float]] = {}`, or `functools.cached_property` for zero-argument computed attributes. The cache dies with the instance.
3. ⚠️ **Not** `self.embed = lru_cache()(self._embed)`. It works, but it creates a cycle (`self → cache wrapper → bound method → self`). Instances are freed only when the cyclic GC eventually runs, not deterministically. Python linters flag the original pattern as `B019` (flake8-bugbear / ruff).

**Related traps in the same family:**

- Module-level `dict` used as a cache with no bound or TTL. Use `cachetools.TTLCache` or an external cache.
- Storing exceptions or tracebacks on long-lived objects (see §1.3). Store `repr(e)` or a formatted string instead.
- Registering instance-bound callbacks on a global registry (signals, event buses) without `weakref.WeakMethod`.

---

## 3. Runnable Code Example

This script proves the leak with `weakref` and `tracemalloc`, then shows the fix. Save it as `day01_memory.py` and run it.

```python
"""
Day 1 — Proving (and fixing) a memory leak caused by @lru_cache on a method.

Run:  python day01_memory.py        (Python 3.12+)
"""
from __future__ import annotations

import gc
import os
import sys
import tracemalloc
import weakref
from collections.abc import Callable
from functools import lru_cache
from typing import Final, Protocol

PAYLOAD_BYTES: Final = 256_000  # stands in for a heavy client / tokenizer / model handle


class Pricer(Protocol):
    def price(self, sku: str) -> float: ...


class LeakyPricer:
    def __init__(self, region: str) -> None:
        self.region = region
        self.blob = bytearray(PAYLOAD_BYTES)

    @lru_cache(maxsize=1024)  # BUG: `self` is part of every cache key -> the cache owns instances
    def price(self, sku: str) -> float:
        return (hash((self.region, sku)) % 10_000) / 100


@lru_cache(maxsize=4096)
def _price(region: str, sku: str) -> float:
    """Pure function keyed only on small, hashable primitives."""
    return (hash((region, sku)) % 10_000) / 100


class SafePricer:
    def __init__(self, region: str) -> None:
        self.region = region
        self.blob = bytearray(PAYLOAD_BYTES)

    def price(self, sku: str) -> float:
        return _price(self.region, sku)


def refcount_tour() -> None:
    payload = object()
    print("refcount(new object)    :", sys.getrefcount(payload))  # 2 = `payload` + call argument
    alias = payload
    container = [payload]
    print("after alias + list slot :", sys.getrefcount(payload))  # 4
    del alias, container
    print("after del               :", sys.getrefcount(payload))  # back to 2
    print("refcount(None)          :", sys.getrefcount(None), "<- immortal (PEP 683)")
    print("gc thresholds           :", gc.get_threshold())
    print()


def churn(factory: Callable[[str], Pricer], n: int) -> list[weakref.ref[Pricer]]:
    """Simulate request handling: build, use, and drop n instances."""
    refs: list[weakref.ref[Pricer]] = []
    for i in range(n):
        pricer = factory(f"region-{i}")
        pricer.price("sku-42")
        refs.append(weakref.ref(pricer))
    return refs


def measure(label: str, factory: Callable[[str], Pricer], n: int = 200) -> list[weakref.ref[Pricer]]:
    gc.collect()
    tracemalloc.start()
    baseline = tracemalloc.take_snapshot()

    refs = churn(factory, n)
    gc.collect()  # rule out "it's just uncollected cycles"

    alive = sum(r() is not None for r in refs)
    current = tracemalloc.take_snapshot()
    tracemalloc.stop()

    diff = current.compare_to(baseline, "lineno")
    retained = sum(stat.size_diff for stat in diff)
    top = diff[0]
    frame = top.traceback[0]
    print(f"{label:<12} alive={alive:>4}/{n}   retained~{retained / 1e6:6.1f} MB")
    print(f"{'':<12} top site: {os.path.basename(frame.filename)}:{frame.lineno} "
          f"(+{top.size_diff / 1e6:.1f} MB)")
    return refs


def main() -> None:
    refcount_tour()

    leaky_refs = measure("LeakyPricer", LeakyPricer)
    measure("SafePricer", SafePricer)

    print("\nleaky cache:", LeakyPricer.price.cache_info())
    LeakyPricer.price.cache_clear()  # the ONLY thing that releases those instances
    print("after cache_clear, leaky instances alive:", sum(r() is not None for r in leaky_refs))
    print("shared safe cache:", _price.cache_info())


if __name__ == "__main__":
    main()
```

**Expected output (abridged):**

```text
refcount(new object)    : 2
after alias + list slot : 4
after del               : 2
refcount(None)          : 4294967295 <- immortal (PEP 683)

LeakyPricer  alive= 200/200   retained~  51.3 MB
             top site: day01_memory.py:27 (+51.2 MB)
SafePricer   alive=   0/200   retained~   0.1 MB
...
after cache_clear, leaky instances alive: 0
```

`tracemalloc` points to line 27 (`self.blob = bytearray(...)`). That tells you *what* is retained, but not *who* holds it. To find the holder, call `gc.get_referrers(obj)` on one of the surviving instances. You'll find a key tuple `(instance, 'sku-42')`, and following *its* referrers leads to the `lru_cache` wrapper's internal storage.

**Exercises (10 min):**

1. Change `maxsize=1024` to `maxsize=None` and `n` to 2000. Watch the retained memory grow linearly.
2. Replace `SafePricer` with the `self.price = lru_cache()(self._price)` variant. Comment out the `gc.collect()` in `measure` and observe that the instances survive until the next collection.
3. Run `python -X importtime -c "import json"` and `python -X tracemalloc=5 day01_memory.py` to get familiar with the `-X` flags.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `mCoding python garbage collection reference counting` | **mCoding** (James Murphy) | Precise, CPython-source-level explanations; short |
| Deep dive | `Nina Zakharenko Memory Management in Python The Basics PyCon` | **PyCon US** | Clear mental model of names, refcounts, GC generations |
| Production | `memray python memory profiler Pablo Galindo` | **PyCon / EuroPython** | Finding real leaks and native-memory issues in services |

**Tools to bookmark:** `tracemalloc` (stdlib), `memray` (Bloomberg; live, native and flame-graph modes), `objgraph` (reference-chain graphs), `py-spy dump` (inspect a live process without restarting it).

---

[← Week 1](README.md) · [Day 2 — Hashing →](day-02-hashing.md)
