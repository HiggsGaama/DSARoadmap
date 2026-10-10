# Day 17 — Dataclasses, `__slots__` & Pydantic: What Records Really Cost

[← Week 3](README.md) · **Track:** Python Systems · **Time box:** 55 min (concept 20 · gotcha 10 · code 20 · review 5)

> **Why this matters for platform work:** An inference gateway may create millions of request, chunk and embedding records per minute. The choice between a plain class, a dataclass, a `NamedTuple`, a `TypedDict` and a Pydantic model shows up directly in RSS, GC pauses and p99 latency. One mis-specified default can also give every record the same timestamp.

**Prerequisite:** Day 15 (class creation: `@dataclass` is a class decorator that runs after the class exists) and Day 10 (descriptors: `__slots__` are descriptors).

---

## 1. Core Concept

### 1.1 `@dataclass` is a code generator

`@dataclass` runs **once, at class-definition time**. It:

1. Reads `cls.__annotations__` in order and builds a list of `Field` objects (stored in `cls.__dataclass_fields__`). A `ClassVar[...]` annotation is skipped, and `KW_ONLY` is a marker.
2. **Generates Python source text** for `__init__`, `__repr__`, `__eq__` (and `__lt__` etc. with `order=True`, `__hash__` per the rules below), then `exec`s it to create real functions and attaches them to the class.
3. Returns the class. With `slots=True` it returns a **new** class, because `__slots__` can't be added to a class after it's created.

The generated `__init__` is ordinary bytecode with no per-call reflection. That's why constructing a dataclass costs about the same as a hand-written class. The cost is paid once, at import.

Two generated behaviours to remember:

| Options | `__hash__` | Why |
|---|---|---|
| `eq=True` (default), `frozen=False` | **Set to `None` → unhashable** | Mutable objects that compare by value can't be safely used as dict keys |
| `eq=True, frozen=True` | Generated from all fields | Immutable value object, so it's hashable (only if every field is hashable) |
| `unsafe_hash=True` | Forced | Almost always a mistake |

`frozen=True` works by generating a `__setattr__` that raises `FrozenInstanceError`. The generated `__init__` therefore has to bypass it with `object.__setattr__(self, name, value)` for every field, so **frozen construction is noticeably slower** (you'll measure roughly 2–3× below).

### 1.2 `__slots__`: replacing the per-instance dict with fixed offsets

A normal instance stores attributes in a `__dict__`. Since 3.11, CPython stores attribute *values* inline and only materializes a real dict when something asks for `__dict__` (`vars(obj)`, some serializers, debuggers). The savings from slots are therefore smaller than older blog posts claim, but they're real and predictable.

With `__slots__ = ("x", "y", "z")`, the class gets a fixed C-level layout. Each slot becomes a **member descriptor** on the class (Day 10) that reads from and writes to a fixed offset in the instance. The results:

- Less memory per instance, and no `__dict__` that could ever be materialized.
- Faster attribute access (a fixed offset instead of a dict/inline-values lookup).
- Typos fail loudly: `obj.tmeperature = 0.2` → `AttributeError` instead of silently creating a new attribute.

Costs and traps:

- **No `__dict__`**, so `functools.cached_property` doesn't work (it writes to `__dict__`), and neither does adding attributes dynamically.
- **No `__weakref__`** by default. Use `@dataclass(slots=True, weakref_slot=True)` (3.11+) if you need weak references or a `WeakValueDictionary` cache.
- **Inheritance:** every class in the hierarchy must declare slots, or a `__dict__` comes back. Multiple inheritance from two slotted bases with non-empty slots raises `TypeError: multiple bases have instance lay-out conflict`.

### 1.3 Pydantic v2: validation is a different job

A dataclass **trusts** its inputs: `DCPoint("a", None, [])` happily constructs an instance. Pydantic **validates and coerces**. In v2, the schema for each model is compiled when the class is defined (Day 15: `BaseModel`'s metaclass) into a validator tree executed by **`pydantic-core`, written in Rust**.

| API | What it does | Use for |
|---|---|---|
| `Model.model_validate(data)` | Full validation + coercion (lax mode: `"512"` → `512`) | **Every untrusted boundary**: HTTP bodies, queue messages, LLM JSON output |
| `Model.model_validate_json(raw)` | Parse and validate in one Rust pass, without an intermediate dict | Hot JSON endpoints. Faster than `json.loads` + `model_validate` |
| `Model.model_construct(**data)` | **No validation**, just sets attributes | Data you already validated, such as rows from your own DB. Measure first: see below |
| `TypeAdapter(T).validate_python(x)` | Validation for *any* type (`list[int]`, `TypedDict`, dataclass), no model needed | Validating a `TypedDict` or `list[...]` at a boundary without defining a model class |
| `ConfigDict(strict=True)` | Disables coercion | Internal APIs where `"512"` should be an error |

**The rule:** validate **once at the edge**, then pass trusted, cheaper objects (dataclasses, or the same model) inward. Re-validating the same data on every internal hop is a common reason for a latency budget disappearing.

### 1.4 Choosing a record type

| Need | Choose |
|---|---|
| Internal value object, many instances, hashable | `@dataclass(frozen=True, slots=True)` |
| Internal mutable state | `@dataclass(slots=True)` |
| Tuple compatibility (unpacking, used as a `dict` key, `csv` rows) | `NamedTuple` |
| Describing the shape of JSON/dicts **for the type checker only** | `TypedDict` (Day 19). No runtime cost and no runtime checks |
| Untrusted input, coercion, JSON schema / OpenAPI generation | Pydantic `BaseModel` |

---

## 2. Production Gotcha: defaults are evaluated once, at class-definition time

```python
@dataclass
class InferenceJob:
    prompt: str
    created_at: datetime = datetime.now(UTC)      # 🚨 evaluated ONCE, at import
    retry: RetryPolicy = RetryPolicy()            # 🚨 ONE object shared by every job
```

**What goes wrong:** a default expression in a class body is an ordinary expression evaluated when the class body runs, at import time. Every `InferenceJob` gets the **same** `created_at`, the moment the worker process started. That produces SLA dashboards showing every job as 6 hours old, and TTL-based cleanup deleting fresh work. Every job also shares one `RetryPolicy`, so a single request that bumps `job.retry.attempts` changes it for all future requests in that process.

**Why dataclasses don't save you:** `@dataclass` rejects defaults whose type is **unhashable** (`list`, `dict`, `set`). A `datetime` is hashable, and so is any ordinary class instance (identity hash), so both pass silently. Plain classes, plain function arguments (`def f(x=[])`) and SQLAlchemy `Column(default=datetime.now())` (called once!) have the same bug.

**Fixes:**

1. **`field(default_factory=...)`** for anything mutable or time-dependent: `field(default_factory=lambda: datetime.now(UTC))`, `field(default_factory=RetryPolicy)`. In Pydantic: `Field(default_factory=...)`. In SQLAlchemy, pass the callable without calling it: `default=datetime.now`.
2. **Make shared defaults immutable:** `frozen=True` dataclasses and tuples instead of lists.
3. **Remember that `frozen` is shallow.** `@dataclass(frozen=True)` with a `list` field can still be mutated through the list, and `hash()` raises `TypeError: unhashable type: 'list'`, for example the first time it's used as an `lru_cache` key in production. Use `tuple[str, ...]` and `frozenset`.

**Lint for it:** ruff `B006` (mutable argument default), `B008` (function call in argument default) and `RUF009` (function call in a dataclass default) catch most of these.

---

## 3. Runnable Code Example

Save as `day17_records.py`. It measures memory and construction cost for five record types, inspects what `@dataclass` generated, reproduces the default-evaluation gotchas, and (if Pydantic is installed) measures validation cost. Without Pydantic, section 4 is skipped cleanly.

```python
"""
Day 17 - Dataclasses, __slots__, and validation cost: measured, not guessed.

Run:  python day17_records.py        (Python 3.12+; the Pydantic section is optional)
"""
from __future__ import annotations

import dataclasses
import inspect
import time
import timeit
import tracemalloc
from collections.abc import Callable
from dataclasses import KW_ONLY, dataclass, field
from datetime import UTC, datetime
from typing import Any, NamedTuple


# ---------------------------------------------------------------------------
# 1. Five ways to hold three floats
# ---------------------------------------------------------------------------

class PlainPoint:
    def __init__(self, x: float, y: float, z: float) -> None:
        self.x, self.y, self.z = x, y, z


@dataclass
class DCPoint:
    x: float
    y: float
    z: float


@dataclass(slots=True)
class SlotPoint:
    x: float
    y: float
    z: float


@dataclass(frozen=True, slots=True)
class FrozenSlotPoint:
    x: float
    y: float
    z: float


class NTPoint(NamedTuple):
    x: float
    y: float
    z: float


Factory = Callable[[float, float, float], object]
KINDS: list[tuple[str, Factory]] = [
    ("plain class", PlainPoint),
    ("@dataclass", DCPoint),
    ("@dataclass(slots)", SlotPoint),
    ("@dataclass(frozen,slots)", FrozenSlotPoint),
    ("NamedTuple", NTPoint),
]


def bytes_per_instance(factory: Factory, n: int = 100_000) -> float:
    """Traced bytes per object (instance + its __dict__, if any). Coordinates are shared floats."""
    x, y, z = 1.5, 2.5, 3.5
    items: list[object] = [None] * n          # pre-size so list growth isn't measured
    tracemalloc.start()
    start = tracemalloc.get_traced_memory()[0]
    for i in range(n):
        items[i] = factory(x, y, z)
    used = tracemalloc.get_traced_memory()[0] - start
    tracemalloc.stop()
    return used / n


def demo_cost() -> None:
    print(f"{'kind':<26}{'bytes/obj':>10}{'ns/construct':>14}  has __dict__")
    for name, factory in KINDS:
        per_obj = bytes_per_instance(factory)
        ns = timeit.timeit(lambda: factory(1.5, 2.5, 3.5), number=200_000) / 200_000 * 1e9
        has_dict = hasattr(factory(1.5, 2.5, 3.5), "__dict__")
        print(f"{name:<26}{per_obj:>10.0f}{ns:>14.0f}  {has_dict}")


# ---------------------------------------------------------------------------
# 2. What @dataclass generated
# ---------------------------------------------------------------------------

def demo_generated() -> None:
    print("\nsignature:      ", inspect.signature(DCPoint))
    print("fields:         ", [f.name for f in dataclasses.fields(DCPoint)])
    print("__match_args__: ", DCPoint.__match_args__)
    print("__slots__:      ", SlotPoint.__slots__)
    print("DCPoint.__hash__ is None ->", DCPoint.__hash__ is None, "(eq=True without frozen => unhashable)")
    print("frozen hashable ->", hash(FrozenSlotPoint(1.0, 2.0, 3.0)) == hash(FrozenSlotPoint(1.0, 2.0, 3.0)))
    try:
        FrozenSlotPoint(1.0, 2.0, 3.0).x = 9.0  # type: ignore[misc]
    except dataclasses.FrozenInstanceError as exc:
        print("frozen assignment ->", type(exc).__name__, exc)

    @dataclass
    class Base:
        model: str
        temperature: float = 0.0

    @dataclass
    class Request(Base):
        _: KW_ONLY                 # everything below is keyword-only
        prompt: str                # no default -- legal only because it's keyword-only
        max_tokens: int = 256

    print("kw_only:        ", inspect.signature(Request))


# ---------------------------------------------------------------------------
# 3. The gotcha: defaults are evaluated ONCE, at class-definition time
# ---------------------------------------------------------------------------

class RetryPolicy:
    def __init__(self) -> None:
        self.attempts = 3


@dataclass
class BuggyJob:
    name: str
    created_at: datetime = datetime.now(UTC)   # BUG: evaluated once, at import
    retry: RetryPolicy = RetryPolicy()         # BUG: one shared mutable object (accepted: it's hashable)


@dataclass
class Job:
    name: str
    created_at: datetime = field(default_factory=lambda: datetime.now(UTC))
    retry: RetryPolicy = field(default_factory=RetryPolicy)
    tags: list[str] = field(default_factory=list)


@dataclass(frozen=True)
class Prompt:
    template: str
    stop: list[str]                            # BUG: frozen is SHALLOW


def demo_gotchas() -> None:
    print()
    a = BuggyJob("a")
    time.sleep(0.01)
    b = BuggyJob("b")
    print("buggy: same created_at?", a.created_at == b.created_at)
    a.retry.attempts = 10
    print("buggy: b.retry.attempts =", b.retry.attempts, "(mutated through a)")

    c = Job("c")
    time.sleep(0.01)
    d = Job("d")
    c.retry.attempts = 10
    print("fixed: same created_at?", c.created_at == d.created_at, "| d.retry.attempts =", d.retry.attempts)

    try:
        @dataclass
        class Rejected:
            items: list[str] = []          # dataclasses DO catch list/dict/set (unhashable defaults)
    except ValueError as exc:
        print("caught at definition:", exc)

    p = Prompt("Answer: ", ["\n"])
    p.stop.append("###")                   # no error: the list itself is mutable
    print("frozen but mutated:", p.stop)
    try:
        hash(p)
    except TypeError as exc:
        print("hash(frozen Prompt) ->", exc)


# ---------------------------------------------------------------------------
# 4. Optional: what validation costs (Pydantic v2)
# ---------------------------------------------------------------------------

def demo_pydantic() -> None:
    print()
    try:
        from pydantic import BaseModel, TypeAdapter, ValidationError
    except ImportError:
        print("pydantic not installed -> skipping section 4 (pip install pydantic)")
        return

    class EmbedRequest(BaseModel):
        model: str
        inputs: list[str]
        dimensions: int = 1024

    raw: dict[str, Any] = {"model": "embed-v3", "inputs": ["hello"] * 16, "dimensions": "512"}

    req = EmbedRequest.model_validate(raw)
    print("validated dimensions:", repr(req.dimensions), "(lax mode coerced '512' -> 512)")

    trusted = EmbedRequest.model_construct(**raw)
    print("model_construct dimensions:", repr(trusted.dimensions), "(no validation: still a str!)")

    n = 20_000
    t_validate = timeit.timeit(lambda: EmbedRequest.model_validate(raw), number=n) / n * 1e6
    t_construct = timeit.timeit(lambda: EmbedRequest.model_construct(**raw), number=n) / n * 1e6
    t_dataclass = timeit.timeit(lambda: Job("x"), number=n) / n * 1e6
    print(f"per call: validate {t_validate:.1f} us | construct {t_construct:.1f} us | stdlib dataclass {t_dataclass:.1f} us")

    ids = TypeAdapter(list[int])           # validation for any type, no model class needed
    print("TypeAdapter(list[int]):", ids.validate_python(["1", 2, 3.0]))
    try:
        EmbedRequest.model_validate({"model": "embed-v3", "inputs": "not-a-list"})
    except ValidationError as exc:
        print("ValidationError:", exc.error_count(), "error(s) ->", exc.errors()[0]["loc"], exc.errors()[0]["type"])


if __name__ == "__main__":
    demo_cost()
    demo_generated()
    demo_gotchas()
    demo_pydantic()
```

**Expected output** (CPython 3.12 on Windows x64 with Pydantic 2.13; numbers vary by machine and version):

```text
kind                       bytes/obj  ns/construct  has __dict__
plain class                       88           588  True
@dataclass                        88           440  True
@dataclass(slots)                 56           509  False
@dataclass(frozen,slots)          56          1347  False
NamedTuple                        72           662  False

signature:       (x: 'float', y: 'float', z: 'float') -> None
fields:          ['x', 'y', 'z']
__match_args__:  ('x', 'y', 'z')
__slots__:       ('x', 'y', 'z')
DCPoint.__hash__ is None -> True (eq=True without frozen => unhashable)
frozen hashable -> True
frozen assignment -> FrozenInstanceError cannot assign to field 'x'
kw_only:         (model: 'str', temperature: 'float' = 0.0, *, prompt: 'str', max_tokens: 'int' = 256) -> None

buggy: same created_at? True
buggy: b.retry.attempts = 10 (mutated through a)
fixed: same created_at? False | d.retry.attempts = 3
caught at definition: mutable default <class 'list'> for field items is not allowed: use default_factory
frozen but mutated: ['\n', '###']
hash(frozen Prompt) -> unhashable type: 'list'

validated dimensions: 512 (lax mode coerced '512' -> 512)
model_construct dimensions: '512' (no validation: still a str!)
per call: validate 4.1 us | construct 5.2 us | stdlib dataclass 1.4 us
TypeAdapter(list[int]): [1, 2, 3]
ValidationError: 1 error(s) -> ('inputs',) list_type
```

How to read it:

- **Memory:** slots save about 32 bytes per object here (88 → 56). At 10M live objects that's about 320 MB, and the slotted objects also have no `__dict__` that something could materialize later.
- **Frozen construction is roughly 2–3× slower** because of the `object.__setattr__` bypass. Fine for configs and keys; reconsider it for objects created per token.
- **`model_construct` was *slower* than full validation here.** In v2 the validator is compiled Rust, while `model_construct` is pure Python. "Skip validation for speed" advice from the v1 era doesn't automatically apply. Measure before you trade safety for speed.

**Exercises (10 min):**

1. Add a `@functools.cached_property` to `SlotPoint` and access it. Read the error, then fix it by removing `slots=True` (or computing the value eagerly).
2. Add a factory `def touched(x, y, z): o = PlainPoint(x, y, z); vars(o); return o` to `KINDS` and re-measure it. Materializing `__dict__` removes the inline-values savings.
3. If Pydantic is installed, add `model_config = ConfigDict(strict=True)` to `EmbedRequest` and observe what happens to `"dimensions": "512"`.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `ArjanCodes dataclasses` | **ArjanCodes** | Practical tour of `field`, `frozen`, `slots`, `kw_only` and when to use each |
| Deep dive | `mCoding __slots__` | **mCoding** | How slots work at the descriptor/memory-layout level |
| Practical | `Samuel Colvin pydantic v2 rust` | **Conference talks** (PyCon / EuroPython) | Why v2 moved validation into Rust and what that means for performance |

---

[← Day 16 — Tree DFS](day-16-tree-dfs.md) · [Day 18 — Tree BFS →](day-18-tree-bfs.md)
