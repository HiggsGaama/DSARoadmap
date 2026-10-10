# Day 8 — Context Managers & Exception Internals: Cleanup That Actually Happens

[← Week 2](README.md) · **Track:** Python Systems · **Time box:** 55 min (concept 15 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** Connection pools, GPU memory leases, distributed locks, temp files and tracing spans all rely on `with` to guarantee release. A context manager that releases correctly on the happy path but leaks on error passes every test, then drains the pool under real traffic, where errors are common.

**Prerequisite:** Day 3 (generators, `throw()`, `close()`).

---

## 1. Core Concept

### 1.1 What `with` actually compiles to

```python
with EXPR as VAR:
    BODY
```

is semantically:

```python
mgr   = EXPR
enter = type(mgr).__enter__          # looked up on the TYPE, not the instance
exit_ = type(mgr).__exit__
value = enter(mgr)
try:
    VAR = value
    BODY
except BaseException as exc:
    if not exit_(mgr, type(exc), exc, exc.__traceback__):
        raise                        # falsy return -> propagate
    # truthy return -> exception SUPPRESSED; execution continues after the with block
else:
    exit_(mgr, None, None, None)
```

Consequences:

- **`__exit__` returning anything truthy suppresses the exception.** That includes `BaseException` subclasses like `KeyboardInterrupt` and `asyncio.CancelledError`. A careless `return self.ok` or `return True` in `__exit__` silently eats errors, *including cancellation*, which breaks timeouts and graceful shutdown.
- Special methods are looked up on the type. Assigning `obj.__exit__ = ...` on an instance has no effect. Since 3.11, a missing protocol raises `TypeError: '...' object does not support the context manager protocol`.
- `__exit__` runs **after** the exception has left `BODY`. If `__exit__` itself raises, the new exception replaces the original, which survives only as `__context__` (§1.4).

### 1.2 `@contextmanager` is a generator driven by `next()` and `throw()`

`contextlib.contextmanager` wraps your generator in a `_GeneratorContextManager`:

| Event | What the wrapper does | What your generator sees |
|---|---|---|
| `__enter__` | `next(gen)` | Runs up to the `yield`; the yielded value is bound by `as` |
| Body succeeds | `next(gen)` | Resumes after `yield`. If it yields *again*: `RuntimeError("generator didn't stop")` |
| Body raises `exc` | `gen.throw(exc)` | **`exc` is raised at the `yield` line** |
| …generator re-raises `exc` | `__exit__` returns `False` | The exception propagates normally |
| …generator swallows it and returns | `__exit__` returns `True` | **Exception suppressed** |
| …generator raises something else | That exception propagates | The original survives as `__context__` |

So **code after `yield` runs only on success unless it's inside `finally`**. That's today's gotcha.

### 1.3 `contextlib` toolbox

| Tool | Use it for |
|---|---|
| `ExitStack` | A **dynamic** number of context managers (open N files, lease from N pools). Unwinds LIFO. `stack.callback(fn, *args)` registers arbitrary cleanup |
| `ExitStack.pop_all()` | **Ownership transfer**: acquire several resources in `__init__`, and if any acquisition fails the earlier ones are released. On success, move the stack into `self` |
| `AsyncExitStack` | Same idea for `async with` (`enter_async_context`, `push_async_callback`) |
| `suppress(*excs)` | Intentional, narrow suppression: `with suppress(FileNotFoundError): os.remove(p)` |
| `closing` / `aclosing` | Call `.close()` / `.aclose()` on exit (Day 3) |
| `nullcontext(x)` | "Optional" context manager: `cm = lock if threaded else nullcontext()` |
| `ContextDecorator` | Objects from `@contextmanager` can also decorate functions: `@timed()`. They're re-created on each call |

### 1.4 Exception chaining: `__context__`, `__cause__`, `__suppress_context__`

| You write | `__context__` | `__cause__` | `__suppress_context__` | Traceback says |
|---|---|---|---|---|
| `raise X` inside an `except` block | the caught exception | `None` | `False` | "**During handling** of the above exception, another exception occurred" |
| `raise X from e` | `e` | `e` | `True` | "The above exception was the **direct cause** of the following exception" |
| `raise X from None` | the caught exception | `None` | `True` | The original is hidden (it's still on `__context__` for debuggers) |

Use `from e` when you **translate** a low-level error into a domain error (`ValueError` → `ConfigError`), so the root cause stays in logs. Use `from None` only when the original is pure noise, such as a `KeyError` inside a `__getattr__` that raises `AttributeError`. Since 3.11, `e.add_note("tenant=acme shard=3")` attaches context without wrapping the exception.

**`return` / `break` / `continue` inside `finally` discards an in-flight exception.** The function returns normally, as if nothing happened. Python 3.14 (PEP 765) emits a `SyntaxWarning` for this, but 3.12/3.13 stay silent.

### 1.5 `ExceptionGroup` and `except*` (3.11+)

Concurrent code can fail in several ways at once. `asyncio.TaskGroup` collects every child failure into an `ExceptionGroup` (or a `BaseExceptionGroup` if any are non-`Exception`).

```python
try:
    async with asyncio.TaskGroup() as tg: ...
except* TimeoutError as eg:      # eg is an ExceptionGroup of ONLY the TimeoutErrors
    retry_later(eg.exceptions)
except* ValueError as eg:
    reject(eg.exceptions)
# anything unmatched is re-raised as a (smaller) group
```

Each `except*` clause runs **at most once**, with the matching sub-group. Several clauses can run for the same group. `eg.split(pred)` and `eg.subgroup(pred)` do the same thing by hand.

---

## 2. Production Gotcha: `@contextmanager` without `try/finally` leaks on every error

```python
@contextmanager
def leased(pool):
    conn = pool.acquire()
    yield conn
    pool.release(conn)          # 🚨 never runs if the body raises
```

When the body raises, the wrapper calls `gen.throw(exc)`, the exception is raised **at the `yield` line**, and it propagates straight out of the generator. `pool.release` is skipped. Every failed request permanently loses one connection. With a pool of 10, the service stops serving after 10 errors: `TimeoutError: QueuePool limit reached`, while CPU and the database look idle.

Two variants of the same bug:

```python
@contextmanager
def span(name):
    s = tracer.start(name)
    try:
        yield s
    except Exception:
        s.record_error()        # 🚨 no `raise`: the generator returns -> __exit__ returns True
    finally:                    #    -> the caller's exception is SILENTLY SUPPRESSED
        s.end()
```

```python
def handler():
    try:
        return process()
    finally:
        metrics.flush()
        return None             # 🚨 swallows any exception from process()
```

**Rules:**

1. Every `@contextmanager` that acquires something looks like `acquire; try: yield; finally: release`. No exceptions to this rule.
2. If you catch around `yield` to log or annotate, **end with a bare `raise`**.
3. Never `return`, `break` or `continue` inside `finally`.
4. Class-based `__exit__` must `return False` (or `None`) unless suppression is the documented purpose, as with `suppress`.

**A related refactor trap, `TaskGroup` wrapping:** moving from `asyncio.gather` to `TaskGroup` changes the exception type. Code that caught `except TimeoutError:` around the gather now receives an `ExceptionGroup`, and the old handler **silently stops matching**. Convert those handlers to `except*` as part of the same change.

---

## 3. Runnable Code Example

Save as `day08_context.py`. It shows the leak and the fix, a class-based context manager with correct `__exit__`, `ExitStack` ownership transfer, exception chaining, `return`-in-`finally`, and `except*` with `TaskGroup`.

```python
"""
Day 8 — Context managers that actually clean up, ExitStack ownership transfer,
exception chaining, and ExceptionGroup handling.

Run:  python day08_context.py        (Python 3.12+)
"""
from __future__ import annotations

import asyncio
from collections.abc import Iterator
from contextlib import ExitStack, contextmanager, suppress
from dataclasses import dataclass, field
from types import TracebackType
from typing import Self


@dataclass
class Pool:
    name: str
    size: int = 2
    in_use: set[int] = field(default_factory=set)
    _next_id: int = 0

    def acquire(self) -> int:
        if len(self.in_use) >= self.size:
            raise RuntimeError(f"{self.name}: pool exhausted ({len(self.in_use)}/{self.size} leased)")
        self._next_id += 1
        self.in_use.add(self._next_id)
        return self._next_id

    def release(self, conn: int) -> None:
        self.in_use.discard(conn)


# ---------------------------------------------------------------------------
# 1. The leak and the fix
# ---------------------------------------------------------------------------
@contextmanager
def leased_buggy(pool: Pool) -> Iterator[int]:
    conn = pool.acquire()
    yield conn               # BUG: body exception is thrown in HERE and never comes back
    pool.release(conn)


@contextmanager
def leased(pool: Pool) -> Iterator[int]:
    conn = pool.acquire()
    try:
        yield conn
    finally:
        pool.release(conn)   # success, exception, or cancellation


def demo_leak() -> None:
    print("-- @contextmanager with and without try/finally")
    for impl in (leased_buggy, leased):
        pool = Pool(impl.__name__, size=3)
        for _ in range(3):                    # three failing requests
            with suppress(ValueError):
                with impl(pool):
                    raise ValueError("bad input")
        try:
            with impl(pool):
                pass
            print(f"  {pool.name:<13} healthy, leased={len(pool.in_use)}")
        except RuntimeError as e:
            print(f"  {pool.name:<13} {e}")


# ---------------------------------------------------------------------------
# 2. Class-based CM: decide, log, never suppress
# ---------------------------------------------------------------------------
class Transaction:
    def __init__(self, log: list[str]) -> None:
        self.log = log

    def __enter__(self) -> Self:
        self.log.append("BEGIN")
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        tb: TracebackType | None,
    ) -> bool:
        self.log.append("ROLLBACK" if exc_type else "COMMIT")
        return False  # a truthy value here would swallow errors, even CancelledError


def demo_transaction() -> None:
    print("\n-- class-based __exit__")
    log: list[str] = []
    with Transaction(log):
        pass
    with suppress(KeyError), Transaction(log):
        raise KeyError("missing row")
    print("  log:", log)


# ---------------------------------------------------------------------------
# 3. ExitStack: all-or-nothing acquisition with ownership transfer
# ---------------------------------------------------------------------------
class MultiLease:
    """Leases one connection from each pool. If any lease fails, the
    already-acquired ones are released before the exception propagates."""

    def __init__(self, pools: list[Pool]) -> None:
        with ExitStack() as stack:
            self.conns = [stack.enter_context(leased(p)) for p in pools]
            self._stack = stack.pop_all()  # success: cleanup ownership moves to self

    def close(self) -> None:
        self._stack.close()

    def __enter__(self) -> Self:
        return self

    def __exit__(self, *exc_info: object) -> None:
        self.close()


def demo_exit_stack() -> None:
    print("\n-- ExitStack.pop_all()")
    primary, replica, empty = Pool("primary"), Pool("replica"), Pool("empty", size=0)
    with MultiLease([primary, replica]) as ml:
        print("  leased:", ml.conns, "| primary in use:", len(primary.in_use))
    print("  after exit, primary in use:", len(primary.in_use))
    try:
        MultiLease([primary, replica, empty])
    except RuntimeError as e:
        print(f"  failed: {e}")
        print("  rolled back -> primary:", len(primary.in_use), "replica:", len(replica.in_use))


# ---------------------------------------------------------------------------
# 4. Exception chaining and the return-in-finally trap
# ---------------------------------------------------------------------------
class ConfigError(Exception):
    pass


def load_workers(raw: str) -> int:
    try:
        return int(raw)
    except ValueError as e:
        raise ConfigError(f"WORKERS must be an int, got {raw!r}") from e


def swallowing_handler() -> str:
    try:
        raise ConnectionError("db down")
    finally:
        return "ok"  # noqa: B012  -- discards the in-flight ConnectionError


def demo_chaining() -> None:
    print("\n-- exception chaining")
    try:
        load_workers("eight")
    except ConfigError as e:
        e.add_note("source=env var WORKERS")
        print("  explicit: cause =", repr(e.__cause__), "| suppress_context =", e.__suppress_context__)
        print("  notes   :", e.__notes__)
    try:
        try:
            {}["missing"]
        except KeyError:
            raise RuntimeError("fallback failed")
    except RuntimeError as e:
        print("  implicit: context =", repr(e.__context__), "| cause =", e.__cause__)
    print("  return-in-finally result:", swallowing_handler(), "(ConnectionError vanished)")


# ---------------------------------------------------------------------------
# 5. TaskGroup -> ExceptionGroup -> except*
# ---------------------------------------------------------------------------
async def call(name: str, fail: type[Exception] | None = None) -> str:
    if fail is not None:
        raise fail(f"{name} failed")
    await asyncio.sleep(0.1)
    return name


async def demo_exception_groups() -> None:
    print("\n-- ExceptionGroup / except*")
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(call("search", TimeoutError))
    except TimeoutError:
        print("  never printed: TaskGroup wraps errors")
    except ExceptionGroup as eg:
        print(f"  plain 'except TimeoutError' missed it; got {eg!r}")

    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(call("embed"))                   # cancelled when siblings fail
            tg.create_task(call("search", TimeoutError))
            tg.create_task(call("rerank", ValueError))
    except* TimeoutError as eg:
        print("  retryable  :", [str(e) for e in eg.exceptions])
    except* ValueError as eg:
        print("  bad request:", [str(e) for e in eg.exceptions])


if __name__ == "__main__":
    demo_leak()
    demo_transaction()
    demo_exit_stack()
    demo_chaining()
    asyncio.run(demo_exception_groups())
```

**Expected output:**

```text
-- @contextmanager with and without try/finally
  leased_buggy  leased_buggy: pool exhausted (3/3 leased)
  leased        healthy, leased=0

-- class-based __exit__
  log: ['BEGIN', 'COMMIT', 'BEGIN', 'ROLLBACK']

-- ExitStack.pop_all()
  leased: [1, 1] | primary in use: 1
  after exit, primary in use: 0
  failed: empty: pool exhausted (0/0 leased)
  rolled back -> primary: 0 replica: 0

-- exception chaining
  explicit: cause = ValueError("invalid literal for int() with base 10: 'eight'") | suppress_context = True
  notes   : ['source=env var WORKERS']
  implicit: context = KeyError('missing') | cause = None
  return-in-finally result: ok (ConnectionError vanished)

-- ExceptionGroup / except*
  plain 'except TimeoutError' missed it; got ExceptionGroup('unhandled errors in a TaskGroup', [TimeoutError('search failed')])
  retryable  : ['search failed']
  bad request: ['rerank failed']
```

Three failed requests leaked all three connections, so the fourth (healthy) request can't get one.

**Exercises (10 min):**

1. In `leased`, add `except Exception: print("logged")` without re-raising. Run `demo_leak` and confirm that `suppress(ValueError)` is no longer needed, because the context manager is now swallowing errors.
2. Make `Transaction.__exit__` return `True`, then wrap a `with Transaction(...)` around `await asyncio.sleep(10)` inside `asyncio.timeout(0.1)`. Explain why the timeout no longer works.
3. Rewrite `MultiLease` *without* `ExitStack` so it still releases correctly on partial failure. Count the nested `try` blocks.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `anthonywritescode contextmanager ExitStack` | **anthonywritescode** | Short, exact explanations of `contextlib` internals and the suppression semantics |
| Deep dive | `mCoding context managers python` | **mCoding** | How `with` desugars; class-based vs generator-based |
| Exceptions | `Irit Katriel exception groups except star PyCon` | **PyCon US** (Irit Katriel, PEP 654 author) | The design rationale and semantics of `ExceptionGroup` / `except*` from its author |
| Practical | `ArjanCodes context managers` | **ArjanCodes** | Structuring resource handling in real applications |

---

[← Day 7 — Consolidation](../week-01/day-07-consolidation.md) · [Day 9 — Monotonic Stack →](day-09-monotonic-stack.md)
