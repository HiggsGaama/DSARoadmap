# Day 15 — Metaclasses & the Class-Creation Pipeline

[← Week 3](README.md) · **Track:** Python Systems · **Time box:** 55 min (concept 20 · gotcha 10 · code 20 · review 5)

> **Why this matters for platform work:** SQLAlchemy models, Pydantic `BaseModel`, Django models, `enum.Enum`, `abc.ABC` and `typing.Protocol` all run code *when your class is defined*. If you understand the pipeline, framework "magic" stops being a black box, you can build plugin registries for model backends and tool handlers, and you can read errors like `metaclass conflict` without searching Stack Overflow.

**Prerequisite:** Day 10 (descriptors and attribute lookup).

---

## 1. Core Concept

### 1.1 A `class` statement is a function call

```python
class Order(Base, route="/orders"):
    qty = Positive()
```

Roughly, this executes:

```python
meta = type(Base)                                            # 0. pick the metaclass
ns = meta.__prepare__("Order", (Base,), route="/orders")     # 1. create the namespace
exec(<class body code>, globals(), ns)                       # 2. run the body INTO ns
Order = meta("Order", (Base,), ns, route="/orders")          # 3-5. build the class object
```

The final call, `meta(...)`, is just `type.__call__`, which calls `meta.__new__` and then `meta.__init__`. **`type` is the default metaclass.** Every ordinary class is an instance of `type`, and `type(name, bases, ns)` builds a class dynamically.

### 1.2 The hooks, in execution order

| # | Hook | Defined on | Runs | Typical use |
|---|---|---|---|---|
| 0 | Metaclass resolution | — | Before anything else | Picks the **most derived** metaclass among the explicit `metaclass=` and the metaclasses of all bases. If none is a subclass of all the others → `TypeError: metaclass conflict` |
| 1 | `__prepare__(name, bases, **kw)` | metaclass (classmethod) | Before the body | Return a custom mapping for the body to write into (reject duplicates, record order, auto-generate values as `enum` does) |
| 2 | Class body | — | Like a function body | Top to bottom; every `def` and assignment becomes a key in the namespace |
| 3 | `Meta.__new__(mcls, name, bases, ns, **kw)` | metaclass | After the body | Inspect or rewrite the namespace before the class exists (ORMs collect columns here) |
| 4a | `__set_name__(owner, name)` | **each attribute** in the namespace | Inside `type.__new__` | Descriptors learn which attribute name they were assigned to |
| 4b | `__init_subclass__(cls, **kw)` | the **parent** class | Inside `type.__new__`, after 4a | Validate or register subclasses and consume class keyword arguments. **This is the 90% replacement for metaclasses** |
| 5 | `Meta.__init__(cls, name, bases, ns, **kw)` | metaclass | After `__new__` returns | Post-creation setup; rarely needed |
| — | `Meta.__call__(cls, *a, **kw)` | metaclass | Every time you **instantiate** the class (`Order(3)`) | Controls `__new__` + `__init__` of *instances*: singletons, instance caching. Usually a bad idea |

Points that matter in practice:

- **`__init_subclass__` runs on the parent and never for the class that defines it.** `Base.__init_subclass__` fires for `Order`, but not for `Base` itself. That's exactly what a registry wants: the abstract base doesn't register itself.
- **Class keyword arguments** (`route="/orders"`) flow to `__prepare__`, `Meta.__new__`, `Meta.__init__` and `__init_subclass__`. If none of them consumes a keyword, `object.__init_subclass__` raises `TypeError` for it.
- **Class decorators** run *after* step 5 on the finished class. `@dataclass` is a decorator, not a metaclass, which is why it composes with any base class.
- Everything here happens **at import time**. An exception in any hook means the module fails to import.

### 1.3 The decision ladder: when (not) to write a metaclass

Use the **least powerful tool** that works:

1. **Class decorator.** Transform one class explicitly (`@dataclass`, `@register("embed")`). Visible at the definition site and doesn't affect subclasses.
2. **`__init_subclass__`.** Apply logic to *every subclass* automatically: registries, validating that required attributes exist, consuming class kwargs.
3. **`__set_name__` on descriptors.** Per-attribute behaviour that needs to know the attribute's own name (fields, validators, ORM columns).
4. **Metaclass.** Only when you need what the others can't do: a custom **namespace** (`__prepare__`), behaviour on the **class object itself** (`__getitem__`, `__iter__`, `__len__`, `__instancecheck__` on the class, as `Enum` does), or control over instantiation (`__call__`).

Metaclasses are inherited by every subclass and **don't compose**, which leads to the gotcha below.

---

## 2. Production Gotcha: metaclass conflicts when mixing frameworks

```python
class Tool(SQLModelBase, abc.ABC):          # ORM metaclass + ABCMeta
    ...
# TypeError: metaclass conflict: the metaclass of a derived class must be a
# (non-strict) subclass of the metaclasses of all its bases
```

**What goes wrong:** every class has exactly one metaclass, and Python must find one that is a subclass of *all* the bases' metaclasses. `abc.ABC` uses `ABCMeta`, `typing.Protocol` uses `_ProtocolMeta` (a subclass of `ABCMeta`), `enum.Enum` uses `EnumType`, and Pydantic, SQLAlchemy and Django each bring their own. Combining two unrelated ones fails **at import time**, often far from the code that changed: someone adds `abc.ABC` to a shared mixin, and every model inheriting it breaks.

Some frameworks pre-combine metaclasses for you (Pydantic's metaclass derives from `ABCMeta`, so `BaseModel` + `abc.ABC` works). Don't count on it for every pair.

**Fixes, in order of preference:**

1. **Don't use an ABC as the mixin.** Describe the interface with a `typing.Protocol` that the class satisfies **without inheriting from it** (Day 19), or use plain duck typing plus a test.
2. **Make the shared behaviour metaclass-free.** Move registry and validation logic into `__init_subclass__`. It composes through cooperative `super().__init_subclass__(**kwargs)` calls, the same way `__init__` does.
3. **Combine the metaclasses explicitly:** `class ModelABCMeta(FrameworkMeta, abc.ABCMeta): pass`, then `metaclass=ModelABCMeta`. This works only if both metaclasses call `super()` cooperatively, and it couples you to the framework's internals.

**Related traps in the same family:**

- **Registry is empty in production:** subclasses register when their *module is imported*. If nothing imports `plugins/embed.py`, the handler doesn't exist. Import plugin packages explicitly at startup, or use `importlib.metadata.entry_points()`.
- **Duplicate registration:** a module imported under two names (`app.handlers` and `handlers` because of a bad `sys.path` entry) defines the class twice. Fail loudly on duplicates, as in the code below.
- **Metaclass `__call__` singletons** break tests (state leaks between tests), pickling and subclassing. Use a module-level instance or dependency injection instead.

---

## 3. Runnable Code Example

Save as `day15_classes.py`. It traces every hook in order, builds a validating descriptor using `__set_name__`, implements a plugin registry with `__init_subclass__`, and reproduces and fixes a metaclass conflict.

```python
"""
Day 15 - The class-creation pipeline, traced; registries without metaclasses.

Run:  python day15_classes.py        (Python 3.12+)
"""
from __future__ import annotations

import abc
import inspect
from typing import Any, ClassVar


# ---------------------------------------------------------------------------
# 1. Trace every hook that runs when a `class` statement executes
# ---------------------------------------------------------------------------

class StrictNamespace(dict[str, Any]):
    """Class-body namespace that rejects accidental redefinition of a name."""

    def __setitem__(self, key: str, value: Any) -> None:
        if key in self and not key.startswith("__"):
            raise TypeError(f"{key!r} defined twice in class body")
        super().__setitem__(key, value)


class TracingMeta(type):
    @classmethod
    def __prepare__(mcls, name: str, bases: tuple[type, ...], /, **kwargs: Any) -> StrictNamespace:
        print(f"  1. Meta.__prepare__      {name!r} kwargs={kwargs}")
        return StrictNamespace()

    def __new__(mcls, name: str, bases: tuple[type, ...], ns: dict[str, Any], /, **kwargs: Any) -> TracingMeta:
        print(f"  3. Meta.__new__ (start)  {name!r} body names={[k for k in ns if not k.startswith('__')]}")
        cls = super().__new__(mcls, name, bases, dict(ns), **kwargs)  # 4a + 4b happen in here
        print(f"  3. Meta.__new__ (end)    {name!r}")
        return cls

    def __init__(cls, name: str, bases: tuple[type, ...], ns: dict[str, Any], /, **kwargs: Any) -> None:
        print(f"  5. Meta.__init__         {name!r}")
        super().__init__(name, bases, ns)


class Positive:
    """Validating descriptor; learns its own attribute name via __set_name__."""

    def __set_name__(self, owner: type, name: str) -> None:
        print(f"  4a. __set_name__         {owner.__name__}.{name}")
        self.public, self.private = name, f"_{name}"

    def __get__(self, obj: object, objtype: type | None = None) -> Any:
        return self if obj is None else getattr(obj, self.private)

    def __set__(self, obj: object, value: float) -> None:
        if value <= 0:
            raise ValueError(f"{self.public} must be > 0, got {value}")
        setattr(obj, self.private, value)


class Base(metaclass=TracingMeta):
    def __init_subclass__(cls, /, route: str = "", **kwargs: Any) -> None:
        super().__init_subclass__(**kwargs)
        print(f"  4b. __init_subclass__    {cls.__name__} route={route!r}")


def demo_pipeline() -> None:
    print("defining Order:")

    class Order(Base, route="/orders"):
        print("  2. class body executing (a plain function call, top to bottom)")
        qty = Positive()

        def __init__(self, qty: int) -> None:
            self.qty = qty

    order = Order(3)
    print("order.qty =", order.qty)
    try:
        Order(0)
    except ValueError as exc:
        print("validation:", exc)

    try:
        class Dup(Base):
            def run(self) -> None: ...
            def run(self) -> None: ...  # type: ignore[no-redef]  # noqa: F811
    except TypeError as exc:
        print("StrictNamespace caught:", exc)


# ---------------------------------------------------------------------------
# 2. Plugin registry with __init_subclass__ -- no metaclass needed
# ---------------------------------------------------------------------------

class Handler:
    registry: ClassVar[dict[str, type[Handler]]] = {}
    event_type: ClassVar[str]

    def __init_subclass__(cls, /, event_type: str, **kwargs: Any) -> None:
        super().__init_subclass__(**kwargs)
        if (existing := Handler.registry.get(event_type)) is not None:
            raise TypeError(
                f"duplicate handler for {event_type!r}: "
                f"{existing.__module__}.{existing.__qualname__} vs {cls.__module__}.{cls.__qualname__}"
            )
        cls.event_type = event_type
        Handler.registry[event_type] = cls

    def handle(self, payload: dict[str, Any]) -> str:
        raise NotImplementedError


class EmbedHandler(Handler, event_type="embed"):
    def handle(self, payload: dict[str, Any]) -> str:
        return f"embedding {len(payload['inputs'])} inputs"


class ChatHandler(Handler, event_type="chat"):
    def handle(self, payload: dict[str, Any]) -> str:
        return f"chat with {len(payload['messages'])} messages"


def dispatch(event: dict[str, Any]) -> str:
    try:
        handler_cls = Handler.registry[event["type"]]
    except KeyError:
        raise LookupError(f"no handler for {event['type']!r}; registered: {sorted(Handler.registry)}") from None
    return handler_cls().handle(event)


def demo_registry() -> None:
    print("\nregistry:", sorted(Handler.registry))
    print(dispatch({"type": "embed", "inputs": ["a", "b"]}))
    print(dispatch({"type": "chat", "messages": [{"role": "user"}]}))
    try:
        class EmbedV2(Handler, event_type="embed"):
            pass
    except TypeError as exc:
        print("duplicate rejected:", exc)


# ---------------------------------------------------------------------------
# 3. The production gotcha: metaclass conflict
# ---------------------------------------------------------------------------

class RegistryMeta(type):
    """Stands in for a framework metaclass (ORM model, pydantic, Django, ...)."""


class FrameworkModel(metaclass=RegistryMeta):
    pass


class Auditable(abc.ABC):  # metaclass is abc.ABCMeta
    @abc.abstractmethod
    def audit_key(self) -> str: ...


def demo_conflict() -> None:
    print()
    try:
        class Broken(FrameworkModel, Auditable):  # type: ignore[metaclass]
            pass
    except TypeError as exc:
        print("TypeError:", exc)

    # Fix 1: a metaclass deriving from both (only if both metaclasses cooperate via super()).
    class ModelABCMeta(RegistryMeta, abc.ABCMeta):
        pass

    class Fixed(FrameworkModel, Auditable, metaclass=ModelABCMeta):
        def audit_key(self) -> str:
            return "fixed-1"

    print("Fix 1 (combined metaclass):", Fixed().audit_key(), [c.__name__ for c in inspect.getmro(ModelABCMeta)[:3]])

    # Fix 2 (usually better): drop the ABC and describe the interface with a Protocol (Day 19)
    # or plain duck typing -- no metaclass involved at all.


if __name__ == "__main__":
    demo_pipeline()
    demo_registry()
    demo_conflict()
```

**Expected output:**

```text
  1. Meta.__prepare__      'Base' kwargs={}
  3. Meta.__new__ (start)  'Base' body names=[]
  3. Meta.__new__ (end)    'Base'
  5. Meta.__init__         'Base'
defining Order:
  1. Meta.__prepare__      'Order' kwargs={'route': '/orders'}
  2. class body executing (a plain function call, top to bottom)
  3. Meta.__new__ (start)  'Order' body names=['qty']
  4a. __set_name__         Order.qty
  4b. __init_subclass__    Order route='/orders'
  3. Meta.__new__ (end)    'Order'
  5. Meta.__init__         'Order'
order.qty = 3
validation: qty must be > 0, got 0
  1. Meta.__prepare__      'Dup' kwargs={}
StrictNamespace caught: 'run' defined twice in class body

registry: ['chat', 'embed']
embedding 2 inputs
chat with 1 messages
duplicate rejected: duplicate handler for 'embed': __main__.EmbedHandler vs __main__.demo_registry.<locals>.EmbedV2

TypeError: metaclass conflict: the metaclass of a derived class must be a (non-strict) subclass of the metaclasses of all its bases
Fix 1 (combined metaclass): fixed-1 ['ModelABCMeta', 'RegistryMeta', 'ABCMeta']
```

Notice that `Base` itself never triggers `__init_subclass__`, that `__set_name__` (4a) runs *before* `__init_subclass__` (4b) and both run *inside* `type.__new__`, and that `Dup` fails during **body execution** (step 2) because the namespace object rejected the second `run`.

**Exercises (10 min):**

1. Add `__call__` to `TracingMeta` that prints and then delegates to `super().__call__(*args, **kwargs)`. Confirm it runs on `Order(3)`, not at class definition.
2. Build the same `Order` class without a `class` statement: `type("Order2", (Base,), {"qty": Positive()}, route="/x")`. Which hooks still fire?
3. Remove `**kwargs` from `Base.__init_subclass__` and pass an unknown class keyword. Read the error and work out which hook raised it.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `mCoding metaclasses python` | **mCoding** | Precise walk through `__prepare__`, `__new__` and `__init_subclass__` with CPython-level detail |
| Deep dive | `James Powell So you want to be a Python expert PyCon 2017` | **PyCon US** (James Powell) | Metaclasses, decorators, generators and context managers shown as one coherent "protocol" model |
| Practical | `anthonywritescode __init_subclass__` | **anthonywritescode** | The lightweight alternative to metaclasses, with real-world usage |

---

[← Day 14 — Consolidation](../week-02/day-14-consolidation.md) · [Day 16 — Tree DFS →](day-16-tree-dfs.md)
