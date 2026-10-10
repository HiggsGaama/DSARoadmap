# Day 21 — Consolidation & Self-Audit

[← Week 3](README.md) · **Track:** Audit · **Time box:** 60 min

| Block | Minutes | Purpose |
|---|---|---|
| A. Retrieval quiz | 14 | 10 questions on this week plus 3 spaced-repetition questions from Weeks 1–2 |
| B. Pattern recognition drill | 8 | DFS vs BFS vs BST reasoning from the problem statement alone |
| C. Production code review | 15 | Find this week's gotchas in a realistic plugin-dispatch handler |
| D. Transfer problem (timed) | 18 | A new BST problem, no hints |
| E. Self-audit & gap log | 5 | Decide what to revisit |

**Rules:** write your answer *before* opening each collapsible. A wrong answer you wrote down teaches more than a right answer you only recognized.

---

## A. Retrieval Quiz (14 min)

### This week

**Q1.** List, in order, what runs when Python executes `class C(Base, metaclass=M, flag=True): x = Field()`.

<details markdown="1"><summary>Answer</summary>

(0) Resolve the most derived metaclass among `M` and `type(Base)`. (1) `M.__prepare__("C", (Base,), flag=True)` returns the namespace. (2) The class body executes into that namespace. (3) `M.__new__` runs, and inside `type.__new__`, (4a) `Field().__set_name__(C, "x")` and then (4b) `Base.__init_subclass__(flag=True)`. (5) `M.__init__`. Class decorators (if any) run after that.

</details>

**Q2.** You need every subclass of `Handler` to register itself under an `event_type` passed as a class keyword. Which hook do you use and why not a metaclass?

<details markdown="1"><summary>Answer</summary>

`__init_subclass__(cls, /, event_type, **kwargs)` on `Handler`. It runs for every subclass but not for `Handler` itself, consumes the class keyword, and composes cooperatively via `super().__init_subclass__(**kwargs)`. A metaclass would be inherited by every subclass and could cause metaclass conflicts with any framework base or ABC it gets mixed with.

</details>

**Q3.** What does `TypeError: metaclass conflict` mean, and what are two ways to fix it?

<details markdown="1"><summary>Answer</summary>

No single metaclass is a subclass of all the bases' metaclasses (e.g. a framework metaclass + `ABCMeta`). Fixes: (1) drop the ABC and use a `Protocol` / duck typing, or move the logic to `__init_subclass__`; (2) define a combined metaclass, `class Combined(FrameworkMeta, ABCMeta)`, and pass `metaclass=Combined`. That works only if both cooperate via `super()`.

</details>

**Q4.** Why is an instance of a plain `@dataclass` unhashable, and when is a dataclass hashable?

<details markdown="1"><summary>Answer</summary>

With `eq=True` (the default) and `frozen=False`, `@dataclass` sets `__hash__ = None`. Mutable objects compared by value would corrupt dicts and sets if they changed after insertion. With `frozen=True` (and `eq=True`), it generates `__hash__` from the fields, and that only works if every field value is itself hashable. A `list` field makes `hash()` raise.

</details>

**Q5.** Give three concrete consequences of `@dataclass(slots=True)`.

<details markdown="1"><summary>Answer</summary>

Any three of: less memory per instance (no `__dict__`); faster attribute access; typos in attribute assignment raise `AttributeError`; `functools.cached_property` stops working (it needs `__dict__`); no weak references unless `weakref_slot=True`; the decorator returns a *new* class object; every class in the hierarchy must use slots, or a `__dict__` comes back.

</details>

**Q6.** `created_at: datetime = datetime.now(UTC)` in a dataclass. What's the bug, why doesn't `@dataclass` reject it, and what's the fix?

<details markdown="1"><summary>Answer</summary>

The default is evaluated **once, when the class body runs at import**, so every instance shares the process start time. `@dataclass` only rejects **unhashable** defaults (`list`, `dict`, `set`), and `datetime` is hashable. Fix: `field(default_factory=lambda: datetime.now(UTC))`.

</details>

**Q7.** When is `Model.model_construct(**data)` appropriate in Pydantic v2, and what did Day 17's measurement show?

<details markdown="1"><summary>Answer</summary>

Only for data that's **already been validated** (e.g. your own DB rows), because it skips validation and coercion entirely. `"512"` stays a `str`. The measurement showed it was *slower* than `model_validate` for a small model, because validation runs in compiled Rust while `model_construct` is pure Python. Measure before trading safety for speed.

</details>

**Q8.** Why does `def top_n[T: HasScore](items: Iterable[T], n: int) -> list[T]` beat `def top_n(items: Iterable[HasScore], n: int) -> list[HasScore]`?

<details markdown="1"><summary>Answer</summary>

The bound type variable keeps the caller's concrete type. Pass `Hit`s and you get `list[Hit]` back, so `.doc_id` type-checks. The protocol-typed version returns `list[HasScore]` and loses every attribute beyond `.score`.

</details>

**Q9.** In Diameter of Binary Tree, why does the recursive helper return one value while updating a different one?

<details markdown="1"><summary>Answer</summary>

The parent can only extend a **straight** branch, so the helper returns its height, `1 + max(left, right)`. The best path *bending* at a node (`left + right`) is a candidate answer that can't be extended upward, so it goes into a separate `nonlocal best`.

</details>

**Q10.** Why does checking each node against its children fail to validate a BST? Give a counter-example.

<details markdown="1"><summary>Answer</summary>

The BST invariant constrains **entire subtrees**, so constraints come from ancestors. Counter-example: `[5, 4, 6, None, None, 3, 7]`. Every parent/child pair is ordered correctly, but `3` sits in the right subtree of `5`. Fix: propagate `(low, high)` bounds down, or check that the in-order sequence is strictly increasing.

</details>

### Spaced repetition (Weeks 1–2)

**Q11.** *(Day 5)* Does `await some_coroutine()` always yield control to the event loop?

<details markdown="1"><summary>Answer</summary>

No. Control returns only when something at the bottom of the await chain yields a pending `Future`. A coroutine that returns without suspending (a cache hit, for example) runs synchronously. Use `await asyncio.sleep(0)` to yield explicitly inside long loops.

</details>

**Q12.** *(Day 10)* Why can a `functools.cached_property` value be overwritten by a plain assignment, while a `property` can't?

<details markdown="1"><summary>Answer</summary>

`property` is a **data descriptor** (it defines `__set__`), and data descriptors take precedence over the instance `__dict__`. `cached_property` is a **non-data descriptor** (only `__get__`). It stores the computed value in the instance `__dict__`, which then wins on later lookups. That's how the caching works, and it's also why `obj.attr = x` shadows it silently and why the value can go stale.

</details>

**Q13.** *(Day 9)* What phrase in a problem statement points to a monotonic stack, and what does the stack hold?

<details markdown="1"><summary>Answer</summary>

"**Next/previous greater (or smaller) element**", "how many days until warmer", "span", "largest rectangle". The stack holds **indices** of elements still waiting for their answer, kept in monotonic order of value. Each new element pops (and resolves) everything it dominates, which is $O(n)$ total because each index is pushed and popped once.

</details>

---

## B. Pattern Recognition Drill (8 min)

For each statement, name the approach (DFS pre-order / DFS post-order (+ global) / BFS / BST walk / in-order) and the **one phrase** that gave it away. Aim for under 30 seconds each.

1. "Return the minimum depth of a binary tree: the number of nodes on the shortest root-to-leaf path."
2. "Determine whether a binary tree is height-balanced."
3. "Return the average value of the nodes on each level."
4. "Given a BST and two nodes, find their lowest common ancestor."
5. "Return the maximum sum of any path between two nodes (values may be negative)."
6. "Return all root-to-leaf paths whose values sum to `target`."
7. "Given a BST, return the sum of values of all nodes with values in `[low, high]`."
8. "Return the values of a binary tree in zigzag level order."

<details markdown="1"><summary>Answers</summary>

| # | Approach | Giveaway |
|---|---|---|
| 1 | **BFS**, returning at the first leaf | "minimum depth", "shortest". BFS stops early; DFS must explore every path |
| 2 | **DFS post-order**, returning height or `-1` as a sentinel | "balanced": a subtree aggregate |
| 3 | **BFS** with the level-size snapshot | "each level" |
| 4 | **BST walk**: the first node with `p <= val <= q` | "BST", "LCA": one path, $O(h)$ |
| 5 | **DFS post-order + global** (LC 124) | "any path between two nodes": the bend vs. branch split, as in Diameter |
| 6 | **DFS pre-order** with a path list (backtracking) | "root-to-leaf paths", "sum to target" |
| 7 | **BST walk / pruned DFS**: skip the left subtree if `val < low`, the right if `val > high` | "BST", "range" |
| 8 | **BFS**, reversing direction on alternate levels | "level order", "zigzag" |

</details>

---

## C. Production Code Review (15 min)

This plugin-dispatch handler for an LLM gateway passed code review and unit tests. In production it shows the following: every request logs the same `received_at`, one tenant's retry change affects all tenants, the cache never hits and in fact raises on every call, a client sending `"max_tokens": "256"` gets a 500, and a request naming the plugin `"Plugin"` crashes with `NotImplementedError`. **Find at least six defects** tied to this week's material, then write the fixed version before opening the answer.

```python
import json
from dataclasses import dataclass
from datetime import UTC, datetime
from functools import lru_cache
from typing import Any, Protocol, TypedDict, runtime_checkable


class PluginMeta(type):
    registry: dict[str, type] = {}

    def __new__(mcls, name, bases, ns):
        cls = super().__new__(mcls, name, bases, ns)
        mcls.registry[name] = cls
        return cls


class Plugin(metaclass=PluginMeta):
    def run(self, req: "Request") -> str:
        raise NotImplementedError


class RetryPolicy:
    def __init__(self) -> None:
        self.attempts = 3


@dataclass(frozen=True)
class Request:
    plugin: str
    prompt: str
    stop: list[str]
    retry: RetryPolicy = RetryPolicy()
    received_at: datetime = datetime.now(UTC)


class RawRequest(TypedDict):
    plugin: str
    prompt: str
    stop: list[str]
    max_tokens: int


@runtime_checkable
class Runner(Protocol):
    def run(self, req: Request) -> str: ...


def parse(body: bytes) -> tuple[Request, RawRequest]:
    raw: RawRequest = json.loads(body)
    return Request(raw["plugin"], raw["prompt"], raw["stop"]), raw


@lru_cache(maxsize=4096)
def render(req: Request) -> str:
    return PluginMeta.registry[req.plugin]().run(req)


def handle(body: bytes, runner: Any = None) -> str:
    req, raw = parse(body)
    if raw["max_tokens"] > 4096:
        raise ValueError("max_tokens too large")
    if isinstance(runner, Runner):
        return runner.run(req)
    return render(req)
```

<details markdown="1"><summary>Defects & fixed version</summary>

| # | Defect | Consequence | Day |
|---|---|---|---|
| 1 | Metaclass used for a registry; registers **every** class including the base `Plugin`, keyed by `__name__` | `"Plugin"` is dispatchable (`NotImplementedError` → 500). Two plugins with the same class name in different modules **silently overwrite** each other. Mixing `Plugin` with an ABC later causes a metaclass conflict | 15 |
| 2 | `stop: list[str]` on a `frozen=True` dataclass | Frozen is shallow and `hash()` raises `TypeError: unhashable type: 'list'`, so `render`'s `lru_cache` fails on **every** call | 17 |
| 3 | `retry: RetryPolicy = RetryPolicy()` | One mutable object shared by every request. Accepted by `@dataclass` because it's hashable | 17 |
| 4 | `received_at: datetime = datetime.now(UTC)` | Evaluated once at import, so every request has the process start time | 17 |
| 5 | `raw: RawRequest = json.loads(body)` | `Any` passes through, nothing is validated, and `"256" > 4096` raises `TypeError`. A missing key raises `KeyError` deep in the handler | 19 |
| 6 | `isinstance(runner, Runner)` with `@runtime_checkable` used as validation | Only checks that a `run` attribute exists. Any object with a `run` of the wrong signature passes | 19 |
| 7 | `runner: Any = None` parameter | Disables checking at the call site. Prefer a real `Optional[Runner]` and let the type checker verify it | 19 |
| bonus | `received_at` taking part in `__eq__`/`__hash__` | Even after fixing the defaults, a per-request timestamp in the cache key means the cache can never hit. Use `field(..., compare=False)` | 17 |

```python
from __future__ import annotations

import json
from dataclasses import dataclass, field
from datetime import UTC, datetime
from functools import lru_cache
from typing import Any, ClassVar


class Plugin:
    registry: ClassVar[dict[str, type[Plugin]]] = {}
    name: ClassVar[str]

    def __init_subclass__(cls, /, name: str, **kwargs: Any) -> None:
        super().__init_subclass__(**kwargs)
        if name in Plugin.registry:                     # fail loudly on collisions
            raise TypeError(f"plugin {name!r} already registered by {Plugin.registry[name].__qualname__}")
        cls.name = name
        Plugin.registry[name] = cls                     # the base class never registers itself

    def run(self, req: Request) -> str:
        raise NotImplementedError


@dataclass(frozen=True)
class RetryPolicy:
    attempts: int = 3


@dataclass(frozen=True, slots=True)
class Request:
    plugin: str
    prompt: str
    stop: tuple[str, ...]                               # immutable -> hashable -> cacheable
    max_tokens: int
    retry: RetryPolicy = field(default_factory=RetryPolicy)
    received_at: datetime = field(default_factory=lambda: datetime.now(UTC), compare=False)


def parse(body: bytes) -> Request:
    raw: object = json.loads(body)                      # object, not Any: forces narrowing
    if not isinstance(raw, dict):
        raise ValueError("body must be a JSON object")
    plugin, prompt, stop, max_tokens = (raw.get(k) for k in ("plugin", "prompt", "stop", "max_tokens"))
    if not isinstance(plugin, str) or plugin not in Plugin.registry:
        raise ValueError(f"unknown plugin {plugin!r}")
    if not isinstance(prompt, str):
        raise ValueError("prompt must be a string")
    if not (isinstance(stop, list) and all(isinstance(s, str) for s in stop)):
        raise ValueError("stop must be a list of strings")
    if not isinstance(max_tokens, int) or isinstance(max_tokens, bool) or not 0 < max_tokens <= 4096:
        raise ValueError("max_tokens must be an int in 1..4096")
    return Request(plugin, prompt, tuple(stop), max_tokens)


@lru_cache(maxsize=4096)
def render(req: Request) -> str:
    return Plugin.registry[req.plugin]().run(req)


def handle(body: bytes) -> str:
    return render(parse(body))


if __name__ == "__main__":
    class Summarize(Plugin, name="summarize"):
        def run(self, req: Request) -> str:
            return f"summary of {req.prompt!r} (stop={list(req.stop)})"

    print(handle(b'{"plugin": "summarize", "prompt": "hi", "stop": ["###"], "max_tokens": 64}'))
    for bad in (b'{"plugin": "summarize", "prompt": "hi", "stop": [], "max_tokens": "64"}',
                b'{"plugin": "Plugin", "prompt": "hi", "stop": [], "max_tokens": 64}'):
        try:
            handle(bad)
        except ValueError as exc:
            print("rejected:", exc)
    print("cache:", render.cache_info())
```

**Output of the fixed version:**

```text
summary of 'hi' (stop=['###'])
rejected: max_tokens must be an int in 1..4096
rejected: unknown plugin 'Plugin'
cache: CacheInfo(hits=0, misses=1, maxsize=4096, currsize=1)
```

This version passes `mypy --strict`. In a real service, `parse` would be a Pydantic `TypeAdapter` or model (Day 17). The hand-written version shows exactly what has to be checked.

</details>

---

## D. Transfer Problem — LeetCode 230: Kth Smallest Element in a BST (Medium) · 18 min timed

No hints this time. Identify the pattern yourself.

**Statement:** Given the `root` of a binary search tree and an integer `k`, return the `k`-th smallest value (1-indexed) among all node values.

```text
Input:  root = [3, 1, 4, None, 2], k = 1
Output: 1

Input:  root = [5, 3, 6, 2, 4, None, None, 1], k = 3
Output: 3
```

**Constraints:** `1 <= k <= n <= 10^4`, `0 <= Node.val <= 10^4`.

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Pattern

**In-order traversal of a BST yields sorted order** (Day 20, view 2), combined with **iterative DFS with an explicit stack** (Day 16) so you can stop the moment you've seen `k` values.

### Intuition

1. Sorting all values costs $O(n \log n)$, and collecting the full in-order list costs $O(n)$ time and space. Both ignore that you only need the first `k` values.
2. In-order = go left as far as possible, visit, then go right. With an explicit stack: push the whole left spine, pop (that's the next smallest), then move into the popped node's right subtree and repeat.
3. Count the pops. The `k`-th pop is the answer, and you return immediately.

### Complexity

- **Time:** $O(h + k)$. You descend one left spine ($h$) and then perform $k$ pops, each amortized $O(1)$.
- **Space:** $O(h)$ for the stack.

### Code

```python
from __future__ import annotations

from collections import deque


class TreeNode:
    def __init__(self, val: int = 0, left: TreeNode | None = None, right: TreeNode | None = None) -> None:
        self.val = val
        self.left = left
        self.right = right


def build_tree(values: list[int | None]) -> TreeNode | None:
    """LeetCode-style level-order list -> tree. `None` marks a missing child."""
    if not values or values[0] is None:
        return None
    root = TreeNode(values[0])
    queue = deque([root])
    i = 1
    while queue and i < len(values):
        node = queue.popleft()
        for side in ("left", "right"):
            if i < len(values) and (v := values[i]) is not None:
                child = TreeNode(v)
                setattr(node, side, child)
                queue.append(child)
            i += 1
    return root


class Solution:
    def kthSmallest(self, root: TreeNode | None, k: int) -> int:
        stack: list[TreeNode] = []
        node = root
        while stack or node:
            while node:                  # descend left: smaller values first
                stack.append(node)
                node = node.left
            node = stack.pop()           # next value in sorted order
            k -= 1
            if k == 0:
                return node.val          # stop early: no need to visit the rest
            node = node.right            # then the right subtree's values
        raise ValueError("k exceeds the number of nodes")


if __name__ == "__main__":
    s = Solution()
    assert s.kthSmallest(build_tree([3, 1, 4, None, 2]), 1) == 1
    assert s.kthSmallest(build_tree([5, 3, 6, 2, 4, None, None, 1]), 3) == 3
    assert s.kthSmallest(build_tree([5, 3, 6, 2, 4, None, None, 1]), 6) == 6
    assert s.kthSmallest(build_tree([1]), 1) == 1
    try:
        s.kthSmallest(build_tree([2, 1]), 3)
    except ValueError:
        pass
    else:
        raise AssertionError("expected ValueError")
    print("all tests passed")
```

**Follow-up (asked in interviews):** *"The BST is modified often and you need `kthSmallest` frequently. How would you optimize?"* Store the **subtree size** in each node. At each node, compare `k` with `size(left) + 1` and go left, return, or go right with `k − size(left) − 1`. That's $O(h)$ per query, and $O(\log n)$ if the tree is balanced. Database "order statistic trees" and rank queries in sorted sets work this way.

**Self-grade:**

- ✅ Solved in 18 min with the iterative early-exit version → mastered.
- 🟡 Solved by building the full sorted list → correct but not optimal. Re-do it iteratively on Day 28.
- 🔴 Not solved → redo Day 20's in-order version and Day 16's iterative traversal tomorrow, before Week 4.

</details>

---

## E. Self-Audit & Gap Log (5 min)

Rate each item **1–5**: 1 = can't explain, 3 = can explain with notes, 5 = can teach it and spot it in code review.

| Area | Skill | Score |
|---|---|---|
| Class creation | Order of `__prepare__` → body → `__new__` → `__set_name__` → `__init_subclass__` → `__init__` | ☐ |
| Class creation | Choose decorator vs `__init_subclass__` vs descriptor vs metaclass | ☐ |
| Class creation | Diagnose and fix a metaclass conflict | ☐ |
| Records | What `@dataclass` generates; hashing rules; `frozen` is shallow | ☐ |
| Records | `slots=True` savings and costs; measure with `tracemalloc` | ☐ |
| Records | Defaults evaluated at import; `default_factory`; validate once at the edge | ☐ |
| Typing | Protocol vs ABC; limits of `runtime_checkable` | ☐ |
| Typing | PEP 695 generics, bounds, `Self`, variance rules | ☐ |
| Typing | `ParamSpec` decorators; `@overload`; `Any` leaking from boundaries | ☐ |
| DSA | Tree DFS: return value vs global update; iterative conversion | ☐ |
| DSA | Tree BFS: level-size snapshot; when BFS beats DFS | ☐ |
| DSA | BST: bounds propagation; in-order sortedness; early-exit traversal | ☐ |

**Action rules:**

- Any **≤ 2** → redo that day's code example from a blank file on Day 22, before new material.
- Any **3** → add it to `GAPS.md` with a re-test date on Day 28.
- All **≥ 4** → stretch: solve LC 124 (Binary Tree Maximum Path Sum) this weekend.

**Retro (write 3 lines in `GAPS.md`):**

```text
Most surprising thing this week:
One piece of framework "magic" I can now explain:
One thing I'll do differently in Week 4:
```

---

[← Day 20 — BST Invariants](day-20-bst.md) · [Week 3 overview](README.md) · [Day 22 — Cancellation & Structured Concurrency →](../week-04/day-22-cancellation-structured-concurrency.md)
