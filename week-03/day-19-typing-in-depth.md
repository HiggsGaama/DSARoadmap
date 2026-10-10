# Day 19 — Typing in Depth: Protocols, Generics, Variance, ParamSpec

[← Week 3](README.md) · **Track:** Python Systems · **Time box:** 60 min (concept 20 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** Platform code is a library that other teams call: SDK wrappers, model-client abstractions, retry decorators, plugin interfaces. Precise types are the only documentation that's checked automatically. They catch "passed the tenant ID where the request ID goes" before production does. They also have a sharp edge: **types describe, they don't enforce**, and confusing the two causes outages.

**Prerequisite:** Day 15 (`Protocol` and `ABC` are metaclass-based) and Day 17 (`TypedDict` vs Pydantic).

---

## 1. Core Concept

### 1.1 Two separate worlds

| | Static (mypy / pyright, in CI and the editor) | Runtime (CPython) |
|---|---|---|
| Reads annotations? | Yes: the whole point | Stores them in `__annotations__`; **never checks them** |
| `def f(x: int)` called with `"a"` | Error | Runs happily |
| Cost | CI minutes | ~zero (and with `from __future__ import annotations`, annotations are lazy strings; 3.14 makes lazy evaluation the default via PEP 649) |
| Who uses annotations at runtime | — | Libraries that opt in: `dataclasses`, Pydantic, FastAPI, `typing.get_type_hints` |

Everything below is about giving the **static** checker enough information. Runtime safety at boundaries is a separate job (validation, Day 17).

### 1.2 Protocols: structural ("duck") typing that's checked

```python
class Embedder(Protocol):
    dim: int
    def embed(self, texts: list[str]) -> list[list[float]]: ...
```

Any class with a compatible `dim` and `embed` **is** an `Embedder`. It doesn't need to inherit from it, import it, or even know it exists. This is how you write an abstraction over the OpenAI, Anthropic, local and fake embedding clients without forcing them into one hierarchy, and without the metaclass conflicts of Day 15.

- **ABC (nominal):** "is a subclass of" → explicit registration, shared implementation, runtime `isinstance` checks the real inheritance.
- **Protocol (structural):** "has this shape" → best for *consumer-side* interfaces. Define the protocol next to the code that **uses** it, not next to the implementations.
- `@runtime_checkable` enables `isinstance(x, Embedder)`, but it **only checks that the attribute names exist**, not signatures or types (see the code).

### 1.3 Generics with PEP 695 syntax (3.12+)

```python
class LRUCache[K, V]: ...                          # generic class
def first[T](xs: Sequence[T]) -> T: ...            # generic function
def top_n[T: HasScore](xs: Iterable[T], n: int) -> list[T]: ...   # bound: T must have .score
def to_bytes[S: (str, bytes)](s: S) -> S: ...      # constrained: exactly str OR bytes
type JSON = dict[str, JSON] | list[JSON] | str | int | float | bool | None   # type alias
```

**Bound vs. annotating with the protocol directly:** `def top_n(xs: Iterable[HasScore]) -> list[HasScore]` loses information. The caller passed `Hit`s and gets back `HasScore`s, so `.doc_id` becomes an error. The bound type variable `T: HasScore` says the result has *the same type as the input*.

**`Self`** (3.11+) is the return type for fluent/builder methods and alternate constructors. `def where(self, ...) -> Self` keeps `VectorQueryBuilder().where(...).top_k(5)` typed as the subclass.

### 1.4 Variance: why `list[Dog]` isn't a `list[Animal]`

```python
def add_cat(animals: list[Animal]) -> None:
    animals.append(Cat())

dogs: list[Dog] = [Dog()]
add_cat(dogs)              # if this were allowed, `dogs` would now contain a Cat
```

| Variance | Meaning | Examples |
|---|---|---|
| **Invariant** | `C[Dog]` and `C[Animal]` are unrelated | `list`, `dict`, `set`: mutable containers (you can both read and write) |
| **Covariant** | `C[Dog]` is a `C[Animal]` | `Sequence`, `Mapping` (values), `Iterable`, `frozenset`, `tuple`: read-only |
| **Contravariant** | `C[Animal]` is a `C[Dog]` | `Callable[[Animal], None]` can be used where `Callable[[Dog], None]` is expected (argument types) |

**Practical rule:** **accept** the most general read-only type (`Iterable[T]`, `Sequence[T]`, `Mapping[K, V]`) and **return** concrete types (`list[T]`, `dict[K, V]`). With PEP 695 syntax, the checker *infers* the variance of your own generic classes from how the type parameter is used.

### 1.5 ParamSpec, `@overload`, and `TypedDict`

- **`ParamSpec` (`[**P, R]`)** captures an entire parameter list. A decorator typed `Callable[P, R] -> Callable[P, R]` keeps the wrapped function's **exact signature**: keyword names, defaults and return type. Without it, decorators type as `Callable[..., Any]` and silently turn off checking for every decorated function, which matters for retry, caching, tracing and auth decorators.
- **`@overload`** expresses "the return type depends on the arguments", e.g. `lookup(key) -> bytes | None` but `lookup(key, default: bytes) -> bytes`. The stubs are for the checker only, and the single real implementation comes last.
- **`TypedDict`** types dict-shaped data (JSON payloads, LLM messages) with per-key types, plus `NotRequired[...]` / `Required[...]` for optional keys. At runtime it's a **plain `dict`**: no class instance, no validation, no cost.

### 1.6 Making it stick: `mypy --strict` in CI

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true                 # disallow untyped defs, Any generics, implicit re-exports, ...
warn_unreachable = true
[[tool.mypy.overrides]]
module = ["legacy_pkg.*"]     # ratchet: strict for new code, relaxed for legacy
disallow_untyped_defs = false
```

Run it as a required CI check, alongside `ruff`. Adopt it incrementally: strict per package, with an overrides list that only ever shrinks.

---

## 2. Production Gotcha: `Any` at the boundary silently disables checking

```python
class ChatMessage(TypedDict):
    role: Literal["system", "user", "assistant"]
    content: str

def handle(body: bytes) -> int:
    messages: list[ChatMessage] = json.loads(body)      # 🚨 returns Any -> accepted with no checks
    return sum(len(m["content"]) for m in messages)     # 💥 TypeError / KeyError in prod
```

**What goes wrong:** `json.loads`, `response.json()`, `yaml.safe_load`, `pickle.loads`, `os.environ` values, untyped third-party libraries and `**kwargs: Any` all produce **`Any`**. `Any` is compatible with every type in both directions, so the checker accepts the assignment and then trusts the annotation everywhere downstream. The code type-checks cleanly, CI is green, and the first LLM response containing `"content": null` (or a client sending `"role": "robot"`) crashes the handler deep inside business logic, far from where the bad data came in.

**The illusion:** the annotation `list[ChatMessage]` reads like a guarantee, but it's only a **claim**. `cast(...)` makes the same claim more explicitly, and it's equally unchecked.

**Fixes:**

1. **Validate at the edge, then type.** Convert `Any` into a precise type in exactly one place: a hand-written validator (as in the code below), `pydantic.TypeAdapter(list[ChatMessage]).validate_python(data)`, or `msgspec`. Everything inward is then truly typed.
2. **Treat the parse result as `object`, not `Any`.** `raw: object = json.loads(body)` forces you to narrow with `isinstance` before use, and the checker enforces that.
3. **Turn on `Any` hygiene:** `strict = true` includes `warn_return_any` and `disallow_any_generics`. Consider `disallow_any_explicit` for core packages, and `mypy --any-exprs-report` to find where `Any` comes in.
4. **Don't use `@runtime_checkable` + `isinstance` as validation.** It only checks attribute *names* (see `Impostor` in the code).

**Related traps:** `if TYPE_CHECKING:` imports aren't available at runtime, so `get_type_hints()`, Pydantic or FastAPI may raise `NameError` resolving them. A mutable default is still a mutable default even when it's typed (Day 17). And `# type: ignore` without an error code hides future errors too, so always write `# type: ignore[code]`.

---

## 3. Runnable Code Example

Save as `day19_typing.py`. It passes `mypy --strict` with no issues (verified with mypy 2.4 on CPython 3.12), and running it shows what the checker can and can't protect you from.

```python
"""
Day 19 - Typing in depth: what the checker sees vs. what actually runs.

Run:    python day19_typing.py              (Python 3.12+)
Check:  mypy --strict day19_typing.py       (pip install mypy) -> should report no issues
"""
from __future__ import annotations

import functools
import json
import time
from collections import OrderedDict
from collections.abc import Callable, Iterable
from dataclasses import dataclass
from typing import Any, Literal, NotRequired, Protocol, Self, TypedDict, get_type_hints, overload, runtime_checkable


# ---------------------------------------------------------------------------
# 1. Protocols: structural typing ("has the right shape"), no inheritance needed
# ---------------------------------------------------------------------------

@runtime_checkable
class Embedder(Protocol):
    dim: int

    def embed(self, texts: list[str]) -> list[list[float]]: ...


class FakeEmbedder:                      # never mentions Embedder
    dim = 3

    def embed(self, texts: list[str]) -> list[list[float]]:
        return [[float(len(t)), 0.0, 1.0] for t in texts]


class Impostor:                          # right NAMES, wrong signature
    dim = 3

    def embed(self) -> str:
        return "not vectors"


def index_documents(embedder: Embedder, docs: list[str]) -> int:
    return sum(len(vec) for vec in embedder.embed(docs))


# ---------------------------------------------------------------------------
# 2. PEP 695 generics: type parameters, bounds, and Self
# ---------------------------------------------------------------------------

class LRUCache[K, V]:
    """Generic over key and value types; the checker tracks both per instance."""

    def __init__(self, capacity: int) -> None:
        self._data: OrderedDict[K, V] = OrderedDict()
        self._capacity = capacity

    def get(self, key: K) -> V | None:
        if key not in self._data:
            return None
        self._data.move_to_end(key)
        return self._data[key]

    def put(self, key: K, value: V) -> None:
        self._data[key] = value
        self._data.move_to_end(key)
        if len(self._data) > self._capacity:
            self._data.popitem(last=False)


class HasScore(Protocol):
    @property
    def score(self) -> float: ...


@dataclass(frozen=True, slots=True)
class Hit:
    doc_id: str
    score: float


def top_n[T: HasScore](items: Iterable[T], n: int) -> list[T]:
    """Bound type variable: accepts anything with .score AND returns the SAME type (list[Hit], not list[HasScore])."""
    return sorted(items, key=lambda item: item.score, reverse=True)[:n]


class QueryBuilder:
    def __init__(self) -> None:
        self.clauses: list[str] = []

    def where(self, clause: str) -> Self:        # Self: subclasses keep their own type when chaining
        self.clauses.append(clause)
        return self


class VectorQueryBuilder(QueryBuilder):
    def top_k(self, k: int) -> Self:
        self.clauses.append(f"TOP {k}")
        return self


# ---------------------------------------------------------------------------
# 3. ParamSpec: decorators that preserve the wrapped signature
# ---------------------------------------------------------------------------

def retry[**P, R](attempts: int = 3, delay: float = 0.01) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorator(fn: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(fn)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            for attempt in range(1, attempts + 1):
                try:
                    return fn(*args, **kwargs)
                except ConnectionError:
                    if attempt == attempts:
                        raise
                    time.sleep(delay)
            raise AssertionError("unreachable")
        return wrapper
    return decorator


_calls = {"n": 0}


@retry(attempts=3)
def fetch_embedding(text: str, *, model: str = "embed-v3") -> list[float]:
    _calls["n"] += 1
    if _calls["n"] < 3:
        raise ConnectionError("upstream reset")
    return [0.1, 0.2, float(len(text))]


# ---------------------------------------------------------------------------
# 4. @overload: return type depends on argument types
# ---------------------------------------------------------------------------

_STORE: dict[str, bytes] = {"model:latest": b"v3"}


@overload
def lookup(key: str) -> bytes | None: ...
@overload
def lookup(key: str, default: bytes) -> bytes: ...
def lookup(key: str, default: bytes | None = None) -> bytes | None:
    return _STORE.get(key, default)


# ---------------------------------------------------------------------------
# 5. TypedDict: types for dict-shaped data -- zero runtime checks
# ---------------------------------------------------------------------------

class ChatMessage(TypedDict):
    role: Literal["system", "user", "assistant"]
    content: str
    name: NotRequired[str]


def count_chars(messages: list[ChatMessage]) -> int:
    return sum(len(m["content"]) for m in messages)


def validate_message(raw: object) -> ChatMessage:
    """The boundary check the type checker CAN'T do for you."""
    if not isinstance(raw, dict):
        raise ValueError("message must be an object")
    role, content = raw.get("role"), raw.get("content")
    if role not in ("system", "user", "assistant"):
        raise ValueError(f"invalid role {role!r}")
    if not isinstance(content, str):
        raise ValueError(f"content must be str, got {type(content).__name__}")
    msg: ChatMessage = {"role": role, "content": content}
    if isinstance(name := raw.get("name"), str):
        msg["name"] = name
    return msg


def main() -> None:
    print("-- Protocols")
    print("index_documents(FakeEmbedder):", index_documents(FakeEmbedder(), ["hi", "there"]))
    print("isinstance(FakeEmbedder(), Embedder):", isinstance(FakeEmbedder(), Embedder))
    print("isinstance(Impostor(), Embedder):    ", isinstance(Impostor(), Embedder), "<- only checks names exist")

    print("\n-- Generics")
    cache: LRUCache[str, list[float]] = LRUCache(capacity=2)
    cache.put("a", [1.0])
    cache.put("b", [2.0])
    cache.put("c", [3.0])                       # evicts "a"
    print("cache.get('a'):", cache.get("a"), "| cache.get('c'):", cache.get("c"))
    print("LRUCache.__type_params__:", LRUCache.__type_params__)
    best = top_n([Hit("d1", 0.2), Hit("d2", 0.9), Hit("d3", 0.5)], n=2)
    print("top_n ->", [h.doc_id for h in best], "(typed as list[Hit])")
    q = VectorQueryBuilder().where("tenant = 'acme'").top_k(5)   # .top_k exists because where() returns Self
    print("query clauses:", q.clauses)

    print("\n-- ParamSpec")
    print("fetch_embedding ->", fetch_embedding("hello", model="embed-v3"), f"after {_calls['n']} calls")
    print("wrapped name kept:", fetch_embedding.__name__)

    print("\n-- overload")
    print("lookup('missing')        ->", lookup("missing"))
    print("lookup('missing', b'v1') ->", lookup("missing", b"v1"))

    print("\n-- Types are not checked at runtime")
    untrusted: Any = json.loads('[{"role": "robot", "content": 42}]')
    try:
        count_chars(untrusted)                   # the checker sees Any -> silently accepts
    except TypeError as exc:
        print("count_chars(untrusted) ->", type(exc).__name__, exc)
    try:
        count_chars([validate_message(m) for m in untrusted])
    except ValueError as exc:
        print("validated at boundary ->", exc)
    print("get_type_hints(count_chars):", get_type_hints(count_chars))


if __name__ == "__main__":
    main()
```

**Expected output:**

```text
-- Protocols
index_documents(FakeEmbedder): 6
isinstance(FakeEmbedder(), Embedder): True
isinstance(Impostor(), Embedder):     True <- only checks names exist

-- Generics
cache.get('a'): None | cache.get('c'): [3.0]
LRUCache.__type_params__: (K, V)
top_n -> ['d2', 'd3'] (typed as list[Hit])
query clauses: ["tenant = 'acme'", 'TOP 5']

-- ParamSpec
fetch_embedding -> [0.1, 0.2, 5.0] after 3 calls
wrapped name kept: fetch_embedding

-- overload
lookup('missing')        -> None
lookup('missing', b'v1') -> b'v1'

-- Types are not checked at runtime
count_chars(untrusted) -> TypeError object of type 'int' has no len()
validated at boundary -> invalid role 'robot'
get_type_hints(count_chars): {'messages': list[__main__.ChatMessage], 'return': <class 'int'>}
```

`count_chars(untrusted)` type-checked fine because `untrusted` is `Any`, and then failed at runtime. The validated path fails **at the boundary** with a clear error instead.

**Exercises (10 min):**

1. Run `mypy --strict day19_typing.py`. Then change `top_n`'s signature to `def top_n(items: Iterable[HasScore], n: int) -> list[HasScore]` and add `print(best[0].doc_id)`. Read the new error.
2. Remove `ParamSpec` from `retry` (type `fn` as `Callable[..., R]`). Call `fetch_embedding(123, modle="x")` and confirm that mypy no longer catches either mistake.
3. Change `untrusted: Any` to `untrusted: object` and see what mypy now forces you to write.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `anthonywritescode typing ParamSpec` / `anthonywritescode TypeVar` | **anthonywritescode** | Short, exact explanations of individual typing features |
| Deep dive | `ArjanCodes Protocol vs ABC` | **ArjanCodes** | Design-level guidance on structural vs nominal interfaces |
| Practical | `mCoding generics python typing` | **mCoding** | Variance, bounds and generics without hand-waving |

---

[← Day 18 — Tree BFS](day-18-tree-bfs.md) · [Day 20 — BST Invariants →](day-20-bst.md)
