# Day 10 — Descriptors, Attribute Lookup & MRO: What `obj.attr` Really Does

[← Week 2](README.md) · **Track:** Python Systems · **Time box:** 55 min (concept 20 · gotcha 10 · code 20 · review 5)

> **Why this matters for platform work:** ORMs (SQLAlchemy columns), Pydantic and dataclass fields, `functools.cached_property`, `@property`, methods and `__slots__` are all **descriptors**. When a config field "randomly" takes another tenant's value, a cached value goes stale, or a mixin's `__init__` never runs, the explanation is in the lookup algorithm below.

---

## 1. Core Concept

### 1.1 The descriptor protocol

Any object **stored as a class attribute** whose type defines one or more of these methods is a descriptor:

| Method | Called on | Makes it a… |
|---|---|---|
| `__get__(self, obj, objtype=None)` | `obj.attr` (and `Cls.attr`, with `obj=None`) | descriptor |
| `__set__(self, obj, value)` | `obj.attr = value` | **data** descriptor |
| `__delete__(self, obj)` | `del obj.attr` | **data** descriptor |
| `__set_name__(self, owner, name)` | Once, at class creation (3.6+) | (lets the descriptor learn its attribute name) |

**Descriptors only work as class attributes.** `self.x = property(...)` inside `__init__` just stores a property object in the instance dict. Nothing intercepts the access.

### 1.2 The lookup algorithm: `object.__getattribute__(obj, name)`

```text
1. meta_attr = look up `name` along type(obj).__mro__         (cached per type version tag)
2. if meta_attr is a DATA descriptor (type has __set__ or __delete__):
       return type(meta_attr).__get__(meta_attr, obj, type(obj))   <- wins over instance dict
3. if `name` in obj.__dict__:
       return obj.__dict__[name]
4. if meta_attr is a NON-DATA descriptor (only __get__):
       return type(meta_attr).__get__(meta_attr, obj, type(obj))
5. if meta_attr was found:
       return meta_attr                                         <- plain class attribute
6. raise AttributeError  -> type(obj).__getattr__(obj, name) runs, if defined
```

**Assignment** (`obj.name = v`, via `object.__setattr__`): if a data descriptor is found on the type, call its `__set__`. Otherwise write to `obj.__dict__`. Non-data descriptors don't participate in assignment, which is why an instance attribute can **shadow** a method or a `cached_property`.

`__getattr__` is only a **fallback** for failed lookups and is cheap to define. `__getattribute__` intercepts **every** access, including `self.__dict__` inside itself. It's easy to recurse infinitely and it slows every attribute access.

### 1.3 Where the language already uses descriptors

| Feature | Descriptor kind | Behaviour that follows |
|---|---|---|
| **Functions** (methods) | Non-data | `func.__get__(obj, cls)` returns a **new bound method object on every access**, so `obj.m is obj.m` is `False`. An instance attribute can shadow a method |
| `staticmethod` / `classmethod` | Non-data | `__get__` returns the raw function, or one bound to the class |
| `property` | **Data** (always defines `__set__`, which raises `AttributeError` if there's no setter) | Can't be shadowed by the instance dict, so it's evaluated on every access |
| `functools.cached_property` | **Non-data** | First access computes the value and **stores it in `obj.__dict__` under the same name**. Later accesses hit step 3 and never call the descriptor again. Requires a `__dict__` (fails with `__slots__`). **No lock since 3.12**: concurrent first accesses may compute twice. Invalidate with `del obj.attr` |
| `__slots__` entries | **Data** (`member_descriptor`) | Values live in a fixed C array in the object; there's no per-instance `__dict__` |

**Performance note:** since 3.11 the adaptive interpreter specializes `LOAD_ATTR` per call site (instance value, slot, method, property…) and keys the cache on the type's *version tag*. Mutating a class at runtime (monkeypatching, `setattr(Cls, …)` in hot paths) bumps the version and de-optimizes every specialized site.

### 1.4 MRO, C3 linearization and `super()`

`Cls.__mro__` is computed with **C3 linearization**: a child always comes before its parents, the order of the bases list is preserved, and every class appears exactly once. Inconsistent hierarchies raise `TypeError: Cannot create a consistent method resolution order` at class creation.

`super()` (zero-argument form) uses the compiler-provided `__class__` cell and the first argument. `super().m()` means: **"find `m` in `type(self).__mro__`, starting *after* the class this code was defined in."** It does **not** mean "call my parent". In a diamond, `Retrying.__init__`'s `super()` may call `Metered.__init__`, a class `Retrying` has never heard of.

Cooperative multiple inheritance works only if every class in the chain:

1. calls `super().__init__(**kwargs)` unconditionally;
2. consumes its own keyword-only arguments and forwards the rest;
3. lets `object.__init__` reject any leftovers, which catches typos for free.

---

## 2. Production Gotcha: the descriptor that stores values on itself

```python
class Bounded:
    def __init__(self, lo: float, hi: float) -> None:
        self.lo, self.hi = lo, hi

    def __get__(self, obj, objtype=None):
        return self.value                  # 🚨 one descriptor object per CLASS ...

    def __set__(self, obj, value):
        if not self.lo <= value <= self.hi:
            raise ValueError(value)
        self.value = value                 # 🚨 ... so every instance shares this slot


class GenerationConfig:
    temperature = Bounded(0.0, 2.0)
```

The descriptor is created **once**, when the class body runs, and every instance shares it. `self` inside `__get__`/`__set__` is the *descriptor*, not the instance. The last write wins for all instances. In a multi-tenant LLM gateway, tenant A's request gets tenant B's temperature. Unit tests that build one object per test pass.

The "obvious" fix, `self.values[obj] = value` with a plain dict, causes a **Day 1 leak**: the dict holds every instance forever.

**Correct storage options:**

1. **In the instance's `__dict__`** under the attribute's own name, learned via `__set_name__`. A data descriptor wins over the instance dict, so `__get__` still runs and can read it back with `vars(obj)[self.name]`. This is the standard approach.
2. Under a private name (`_temperature`) via `getattr`/`setattr`, which also works with `__slots__` if you declare the slot.
3. `weakref.WeakKeyDictionary` keyed by instance, but only when instances can't hold a `__dict__` and are hashable.

**Related traps:**

- `cached_property` on a mutable object goes **stale** silently after the inputs change. Use it only for values derived from immutable state, or `del` it on every mutation path.
- Assigning a lambda to an instance attribute with a method's name (`client.send = fake_send` in production code) shadows the method for that instance only, which is a source of "works on one object" bugs.
- Mixins that call `Parent.__init__(self)` explicitly instead of `super().__init__(**kwargs)` skip siblings in the MRO, so some layers never initialize.

---

## 3. Runnable Code Example

Save as `day10_descriptors.py`. It covers the precedence rules, the shared-state bug and its fix, bound methods, `cached_property`, and a cooperative `super()` chain.

```python
"""
Day 10 — Descriptors: lookup precedence, validation done right (and wrong),
bound methods, cached_property, and cooperative super().

Run:  python day10_descriptors.py        (Python 3.12+)
"""
from __future__ import annotations

from functools import cached_property
from typing import Any


# ---------------------------------------------------------------------------
# 1. Lookup precedence: data descriptor > instance dict > non-data > class attr > __getattr__
# ---------------------------------------------------------------------------
class DataDesc:
    def __get__(self, obj: object, objtype: type | None = None) -> str:
        return "data descriptor"

    def __set__(self, obj: object, value: object) -> None:
        raise AttributeError("read-only")


class NonDataDesc:
    def __get__(self, obj: object, objtype: type | None = None) -> str:
        return "non-data descriptor"


class Probe:
    data = DataDesc()
    nondata = NonDataDesc()
    plain = "class attribute"

    def __getattr__(self, name: str) -> str:
        return f"__getattr__({name!r})"


def demo_lookup() -> None:
    print("-- lookup precedence (instance dict has all three names)")
    p = Probe()
    vars(p).update(data="instance dict", nondata="instance dict", plain="instance dict")
    for name in ("data", "nondata", "plain", "missing"):
        print(f"  p.{name:<8} -> {getattr(p, name)}")


# ---------------------------------------------------------------------------
# 2. Validation descriptor: shared-state bug vs per-instance storage
# ---------------------------------------------------------------------------
class SharedStateBounded:
    def __init__(self, lo: float, hi: float) -> None:
        self.lo, self.hi = lo, hi
        self.value = 0.0

    def __get__(self, obj: object, objtype: type | None = None) -> float:
        return self.value                        # BUG: shared by all instances

    def __set__(self, obj: object, value: float) -> None:
        if not self.lo <= value <= self.hi:
            raise ValueError(value)
        self.value = value


class Bounded:
    """Data descriptor; stores per-instance values in obj.__dict__ under its own name."""

    def __init__(self, lo: float, hi: float) -> None:
        self.lo, self.hi = lo, hi
        self.name = "<unbound>"

    def __set_name__(self, owner: type, name: str) -> None:
        self.name = name                         # learned once, at class creation

    def __get__(self, obj: object | None, objtype: type | None = None) -> Any:
        if obj is None:
            return self                          # Cls.attr -> the descriptor (introspection, docs)
        try:
            return vars(obj)[self.name]          # data descriptor wins, so we read the dict ourselves
        except KeyError:
            raise AttributeError(self.name) from None

    def __set__(self, obj: object, value: float) -> None:
        if not self.lo <= value <= self.hi:
            raise ValueError(f"{self.name}={value!r} outside [{self.lo}, {self.hi}]")
        vars(obj)[self.name] = value


class BuggyConfig:
    temperature = SharedStateBounded(0.0, 2.0)

    def __init__(self, temperature: float) -> None:
        self.temperature = temperature


class GenerationConfig:
    temperature = Bounded(0.0, 2.0)
    top_p = Bounded(0.0, 1.0)

    def __init__(self, temperature: float, top_p: float = 1.0) -> None:
        self.temperature = temperature           # routed through Bounded.__set__
        self.top_p = top_p


def demo_validation() -> None:
    print("\n-- validation descriptors")
    tenant_a, tenant_b = BuggyConfig(0.2), BuggyConfig(1.5)
    print(f"  buggy: tenant_a.temperature = {tenant_a.temperature}  (set 0.2, got tenant_b's)")
    a, b = GenerationConfig(0.2), GenerationConfig(1.5)
    print(f"  fixed: a={a.temperature} b={b.temperature}  stored in: {vars(a)}")
    try:
        GenerationConfig(temperature=3.0)
    except ValueError as e:
        print("  rejected:", e)
    print("  class access returns descriptor:", type(GenerationConfig.temperature).__name__)


# ---------------------------------------------------------------------------
# 3. Functions are non-data descriptors -> bound methods
# ---------------------------------------------------------------------------
class Model:
    def predict(self, x: float) -> float:
        return 2 * x


def demo_methods() -> None:
    print("\n-- functions as descriptors")
    m = Model()
    print("  m.predict is m.predict :", m.predict is m.predict)   # new bound method each access
    print("  m.predict == m.predict :", m.predict == m.predict)   # same __func__ and __self__
    bound = Model.__dict__["predict"].__get__(m, Model)           # what the dot operator does
    print("  manual __get__(m)(21)  :", bound(21))
    m.predict = lambda x: -1.0  # type: ignore[method-assign]     # instance dict shadows non-data
    print("  after shadowing        :", m.predict(21))


# ---------------------------------------------------------------------------
# 4. cached_property: compute once, store in __dict__, can go stale
# ---------------------------------------------------------------------------
class Document:
    def __init__(self, text: str) -> None:
        self.text = text
        self.computations = 0

    @cached_property
    def token_count(self) -> int:
        self.computations += 1
        return len(self.text.split())


def demo_cached_property() -> None:
    print("\n-- cached_property")
    d = Document("a b c")
    print(f"  {d.token_count}, {d.token_count} -> computed {d.computations}x;"
          f" in __dict__: {'token_count' in vars(d)}")
    d.text = "a b c d e f"
    print(f"  after text change: {d.token_count}  (stale)")
    del d.token_count                            # invalidation = delete the cached entry
    print(f"  after del        : {d.token_count}  (recomputed, total {d.computations}x)")


# ---------------------------------------------------------------------------
# 5. MRO + cooperative super()
# ---------------------------------------------------------------------------
class Service:
    def __init__(self, **kwargs: Any) -> None:
        super().__init__(**kwargs)               # object.__init__ rejects leftovers (typos)
        self.layers: list[str] = []


class Retrying(Service):
    def __init__(self, *, retries: int = 3, **kwargs: Any) -> None:
        super().__init__(**kwargs)               # next in type(self).__mro__, NOT necessarily Service
        self.layers.append(f"Retrying(retries={retries})")


class Metered(Service):
    def __init__(self, *, metric: str = "svc", **kwargs: Any) -> None:
        super().__init__(**kwargs)
        self.layers.append(f"Metered(metric={metric})")


class EmbeddingClient(Retrying, Metered):
    def __init__(self, *, model: str, **kwargs: Any) -> None:
        super().__init__(**kwargs)
        self.layers.append(f"EmbeddingClient(model={model})")


def demo_mro() -> None:
    print("\n-- MRO and cooperative super()")
    print("  MRO   :", " -> ".join(c.__name__ for c in EmbeddingClient.__mro__))
    client = EmbeddingClient(model="e5-large", retries=5, metric="embed")
    print("  layers:", client.layers)
    try:
        EmbeddingClient(model="e5-large", retires=5)   # typo
    except TypeError as e:
        print("  typo caught:", e)


if __name__ == "__main__":
    demo_lookup()
    demo_validation()
    demo_methods()
    demo_cached_property()
    demo_mro()
```

**Expected output:**

```text
-- lookup precedence (instance dict has all three names)
  p.data     -> data descriptor
  p.nondata  -> instance dict
  p.plain    -> instance dict
  p.missing  -> __getattr__('missing')

-- validation descriptors
  buggy: tenant_a.temperature = 1.5  (set 0.2, got tenant_b's)
  fixed: a=0.2 b=1.5  stored in: {'temperature': 0.2, 'top_p': 1.0}
  rejected: temperature=3.0 outside [0.0, 2.0]
  class access returns descriptor: Bounded

-- functions as descriptors
  m.predict is m.predict : False
  m.predict == m.predict : True
  manual __get__(m)(21)  : 42
  after shadowing        : -1.0

-- cached_property
  3, 3 -> computed 1x; in __dict__: True
  after text change: 3  (stale)
  after del        : 6  (recomputed, total 2x)

-- MRO and cooperative super()
  MRO   : EmbeddingClient -> Retrying -> Metered -> Service -> object
  layers: ['Metered(metric=embed)', 'Retrying(retries=5)', 'EmbeddingClient(model=e5-large)']
  typo caught: object.__init__() takes exactly one argument (the instance to initialize)
```

Read the `layers` order closely. Each `__init__` appends *after* its `super()` call returns, so the list is built in **reverse MRO order**. `Retrying`'s `super()` resolved to `Metered`, not `Service`. The typo's error message from `object.__init__` doesn't name the bad argument, but the failure is loud, which beats silently ignoring `retires=5`.

**Exercises (10 min):**

1. Remove `__set__` from `DataDesc` and re-run `demo_lookup`. Which line changes, and why?
2. Add `__slots__ = ("text", "computations")` to `Document`. Read the error `cached_property` raises.
3. Change `Retrying.__init__` to call `Service.__init__(self, **kwargs)` directly. Which layer disappears from `layers`?

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `mCoding descriptors python` | **mCoding** | How `property`, methods and `__set_name__` are built on the protocol |
| Deep dive | `Raymond Hettinger super considered super PyCon` | **PyCon US** (Raymond Hettinger) | The definitive talk on cooperative multiple inheritance and the MRO |
| Reference | `Raymond Hettinger descriptor howto` | **PyCon / conference recordings** | Pair with the official *Descriptor HowTo Guide* in the Python docs, which has pure-Python equivalents of `property`, `classmethod` and functions |
| Practical | `anthonywritescode cached_property` | **anthonywritescode** | Caching gotchas, including the 3.12 lock removal |

---

[← Day 9 — Monotonic Stack](day-09-monotonic-stack.md) · [Day 11 — Binary Search →](day-11-binary-search.md)
