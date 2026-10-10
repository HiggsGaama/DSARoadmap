# Day 35 — Consolidation & Self-Audit

[← Week 5](README.md) · **Track:** Audit · **Time box:** 60 min

| Block | Minutes | Purpose |
|---|---|---|
| A. Retrieval quiz (this week + spaced repetition) | 14 | Recall without notes, including Weeks 1–4 material you haven't touched in a while |
| B. Pattern recognition drill | 8 | Map problem statements to graph patterns in under 30 s each |
| C. Production code review | 15 | Find this week's gotchas in a realistic FastAPI service |
| D. Transfer problem (timed) | 18 | A new problem with a known pattern, no hints |
| E. Self-audit & gap log | 5 | Decide what to revisit before the final week |

**Rules:** write your answer *before* opening each collapsible. A wrong answer you wrote down teaches more than a right answer you only recognized.

---

## A. Retrieval Quiz (14 min)

### This week

**Q1.** Why does `from app.models import Order` fail with "partially initialized module" in a cycle, while `import app.models` (using `app.models.Order` at call time) often works?

<details markdown="1"><summary>Answer</summary>

The importer inserts the module into `sys.modules` **before** executing its body. In a cycle, the second module finds the first one already in `sys.modules` but only partly executed. `from X import Name` needs `Name` to exist *now*, and it doesn't yet. `import X` only binds the module object. The attribute lookup `X.Order` happens later, at call time, when both modules have finished executing.

</details>

**Q2.** A worker runs fine locally but logs every line twice in production, and a module-level registry has two different instances. The entry point is `python app/worker.py`. What happened?

<details markdown="1"><summary>Answer</summary>

The file ran as `__main__`. Later, something did `import app.worker`, which executed the module **a second time** under its real name, so it got its own globals and attached logging handlers again. Run it with `python -m app.worker` (or a `[project.scripts]` entry point), and keep `__main__` blocks thin.

</details>

**Q3.** Name the three ASGI callables/arguments and two facts about request bodies that a naive handler gets wrong.

<details markdown="1"><summary>Answer</summary>

`scope` (connection metadata), `receive()` (client→app events) and `send(msg)` (app→client events). (1) Bodies arrive as **multiple** `http.request` messages, so you must loop until `more_body` is `False`. (2) `receive` is a **one-shot stream**: middleware that reads it must replay the messages, or the endpoint hangs or sees a truncated body.

</details>

**Q4.** A FastAPI service's latency climbs in steps at about 40 concurrent requests while CPU stays low. All endpoints are `def`, not `async def`. Why?

<details markdown="1"><summary>Answer</summary>

Sync endpoints (and sync dependencies) run in AnyIO's threadpool, which allows **40 tokens per process** by default. Requests beyond that queue invisibly before the endpoint even starts. Move hot paths to async drivers, or raise the limiter deliberately and measure the effect.

</details>

**Q5.** Why is `pickle.loads` on a Redis value potentially remote code execution, and what *besides* security breaks during a rolling deploy?

<details markdown="1"><summary>Answer</summary>

A pickle can contain a `REDUCE` opcode that calls any importable callable (via `__reduce__`), so anyone who can write to the cache can run code on every reader. Separately, pickles store `module.ClassName`. Renaming or moving a class makes old entries unloadable, and old and new pods disagree about the same keys. Use JSON/msgpack/msgspec with a versioned key.

</details>

**Q6.** In Dijkstra with `heapq`, why do you push duplicates instead of updating entries, and what line makes that safe?

<details markdown="1"><summary>Answer</summary>

`heapq` has no decrease-key. You push a new `(dist, node)` whenever you improve a node, and **skip stale pops** with `if d > dist[u]: continue`. Because weights are non-negative, the first non-stale pop of each node carries its final distance.

</details>

**Q7.** Why does two-colour (visited / not visited) DFS give false cycles on directed graphs, and how does Union-Find relate to cycle detection?

<details markdown="1"><summary>Answer</summary>

In a diamond (`A→B→D`, `A→C→D`), D is reached twice without any cycle. You need GRAY (on the current path) vs BLACK (finished), because only an edge to a GRAY node is a back edge. Union-Find detects cycles in **undirected** graphs (`union` returns `False` when both endpoints already share a root). It ignores direction, so it can't replace topo sort for dependencies.

</details>

### Spaced repetition (Weeks 1–4)

**S1 (Week 2: context managers & the GIL).** What does it mean when `__exit__` returns `True`? And why do threads speed up blocking I/O but not pure-Python CPU work on the standard (GIL) build?

<details markdown="1"><summary>Answer</summary>

Returning a truthy value from `__exit__` **suppresses** the exception raised inside the `with` block. A context manager that does this unconditionally silently swallows every error. Threads help with I/O because blocking syscalls release the GIL, so other threads run while one waits. Pure-Python bytecode needs the GIL, so CPU-bound threads take turns rather than run in parallel. Use processes, or the free-threaded build, for that.

</details>

**S2 (Week 3: dataclasses & typing).** Why does `@dataclass class C: items: list[str] = []` raise, what's the fix, and what does a `Protocol` give you that an ABC doesn't?

<details markdown="1"><summary>Answer</summary>

Dataclasses reject mutable defaults (`ValueError: mutable default <class 'list'> for field items is not allowed: use default_factory`), because one list would be shared by every instance. Use `field(default_factory=list)`. A `Protocol` is **structural**: any class with matching methods satisfies it for the type checker, with no inheritance or registration. That lets you type third-party objects you don't control.

</details>

**S3 (Week 4: cancellation & backpressure).** Why must you re-raise `asyncio.CancelledError` if you catch it, and what does a bounded `asyncio.Queue(maxsize=N)` give you that an unbounded one doesn't?

<details markdown="1"><summary>Answer</summary>

Cancellation is delivered as an exception. Swallowing it makes the task keep running after its owner (a `TaskGroup`, a `timeout`, server shutdown) asked it to stop, which breaks timeouts and graceful shutdown. Clean up in `finally`, or re-raise. A bounded queue gives you **backpressure**: `await queue.put()` suspends producers when consumers fall behind, so memory stays bounded and slowness propagates upstream instead of turning into an OOM.

</details>

**S4 (Weeks 2–4: DSA).** For "k largest elements of a stream", which heap and what size? For merging overlapping intervals, what's the first step?

<details markdown="1"><summary>Answer</summary>

A **min**-heap of size **k**. The root is the smallest of the k largest, so evict it when a bigger element arrives. That's $O(n \log k)$. For intervals, **sort by start**, then do one pass that merges each interval into the last merged one whenever `start <= last_end`.

</details>

---

## B. Pattern Recognition Drill (8 min)

For each statement, name the algorithm and the **one phrase** that gave it away.

1. "Given a list of package dependencies `(pkg, depends_on)`, return a valid install order or report that none exists."
2. "Accounts that share any email belong to the same person. Merge them."
3. "Find the minimum time for a message to reach every server. Links have non-negative latencies."
4. "In a grid of 0s and 1s, find the minimum number of steps from the top-left to the bottom-right. Each move costs 1."
5. "Edges are added one by one to an undirected graph. After each addition, report the number of connected components."
6. "Find the cheapest flight from A to B using at most K stops."
7. "Moving between cells costs 0 when the arrow points that way and 1 otherwise. Find the minimum cost."
8. "Given words sorted in an alien language, derive the order of its letters."

<details markdown="1"><summary>Answers</summary>

| # | Algorithm | Giveaway |
|---|---|---|
| 1 | Topological sort (Kahn), with a cycle → "none exists" | "dependencies", "order", "report if none" |
| 2 | Union-Find (or DFS on an email graph) | "share any", which signals transitive grouping |
| 3 | Dijkstra, answer = `max(dist)` | "minimum time", "every", "non-negative" weights |
| 4 | BFS (unweighted) | "each move costs 1" |
| 5 | Union-Find with a component counter | "added one by one", "connected components" (online, undirected) |
| 6 | Bellman-Ford limited to K+1 rounds, or BFS by levels. **Not** plain Dijkstra | "at most K stops" breaks the greedy invariant |
| 7 | 0-1 BFS with a deque | weights only 0 or 1 |
| 8 | Topological sort on letter edges taken from adjacent words | "derive the order" from pairwise comparisons |

</details>

---

## C. Production Code Review (15 min)

This service "works" in staging. In production, workers occasionally crash at boot with an `ImportError`. Requests to `/orders` sometimes hang until timeout. Unrelated requests stall while `/orders` is busy. A security scan flags it as critical, and finance reports totals that are off by a cent. **Find at least seven defects** tied to this week's material, then sketch the fix before opening the answer.

```python
# app/main.py
import json
import pickle
import time

import psycopg
from fastapi import FastAPI, Request

from app.models import Order            # app/models.py does: from app.main import settings

settings = json.load(open("config.json"))
db = psycopg.connect(settings["dsn"])   # gunicorn --preload -w 8

app = FastAPI()
audit_log: list[bytes] = []


class AuditMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] == "http":
            message = await receive()
            audit_log.append(message.get("body", b""))
        await self.app(scope, receive, send)


app.add_middleware(AuditMiddleware)


@app.post("/orders")
async def create_order(request: Request) -> dict:
    session = pickle.loads(bytes.fromhex(request.cookies["session"]))
    payload = await request.json()
    order = Order(total=float(payload["total"]))
    time.sleep(0.2)                                   # crude rate limit for the fraud API
    db.execute("INSERT INTO orders (user_id, total) VALUES (%s, %s)", (session["user"], order.total))
    return json.loads(json.dumps({"order": order.__dict__, "at": order.created_at}, default=str))
```

<details markdown="1"><summary>Defects & fixed version</summary>

| # | Defect | Consequence | Day |
|---|---|---|---|
| 1 | `app.main` ↔ `app.models` circular import | `ImportError: partially initialized module` depending on which module is imported first (some entry points or tests pass, the worker crashes) | 29 |
| 2 | Import-time side effects: reading config + **opening a DB connection at import** | Slow, untestable imports. With `--preload` + fork, **8 workers share one socket**, so responses interleave and the connection gets corrupted | 29 |
| 3 | `open("config.json")` never closed; path is relative to the CWD | FD leak; boot fails when started from another directory | 1 / 29 |
| 4 | Middleware consumes `receive` without replaying it, and only reads the first chunk | The endpoint's `await request.json()` waits for a body that already went to the middleware, so it **hangs**. The audit captures truncated bodies | 31 |
| 5 | `audit_log` is an unbounded module-level list of raw bodies | Memory grows forever (Day 1). PII stored in process memory | 1 / 31 |
| 6 | `pickle.loads` on a **client-controlled cookie** | **Remote code execution** | 33 |
| 7 | Money as `float` | `0.1 + 0.2` errors; totals off by a cent | 33 |
| 8 | `time.sleep` inside `async def` | Blocks the event loop and stalls *every* request in the worker | 5 / 31 |
| 9 | Sync `psycopg` call on the event loop, through one shared connection | Blocks the loop; concurrent requests interleave on one connection | 5 / 31 |
| 10 | `json.loads(json.dumps(..., default=str))` round-trip | Wasted CPU on every response; `default=str` silently stringifies datetimes (naive, no TZ) and any object that leaks in | 33 |

```python
# app/settings.py  (no imports from app.main -> no cycle)
from __future__ import annotations

import json
import os
from functools import cache
from pathlib import Path


@cache
def get_settings() -> dict[str, str]:
    path = Path(os.environ.get("APP_CONFIG", Path(__file__).with_name("config.json")))
    with path.open(encoding="utf-8") as f:
        return json.load(f)


# app/main.py
from contextlib import asynccontextmanager
from decimal import Decimal

import psycopg_pool
from fastapi import FastAPI, Request
from pydantic import BaseModel

from app.auth import verify_signed_session       # HMAC/JWT-signed, JSON payload; no pickle
from app.settings import get_settings


class OrderIn(BaseModel):
    total: Decimal                                # exact money; Pydantic parses "19.99" -> Decimal


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.pool = psycopg_pool.AsyncConnectionPool(get_settings()["dsn"], open=False)
    await app.state.pool.open()                   # per worker, after fork
    yield
    await app.state.pool.close()


app = FastAPI(lifespan=lifespan)
# Audit middleware removed: if needed, write a pure-ASGI middleware that REPLAYS
# the chunks it reads, caps the size it records, and ships to a log sink, not a list.


@app.post("/orders")
async def create_order(request: Request, body: OrderIn) -> dict[str, str]:
    session = verify_signed_session(request.cookies.get("session"))
    await fraud_limiter.acquire()                 # async rate limiter, not time.sleep (Day 38)
    async with request.app.state.pool.connection() as conn:
        await conn.execute(
            "INSERT INTO orders (user_id, total) VALUES (%s, %s)", (session.user_id, body.total)
        )
    return {"status": "created", "total": str(body.total)}
```

</details>

---

## D. Transfer Problem — LeetCode 1631: Path With Minimum Effort (Medium) · 18 min timed

No hints. Identify the pattern yourself.

**Statement:** You're given a `rows × cols` grid `heights`. You start at the top-left cell and want to reach the bottom-right cell, moving up, down, left or right. A route's **effort** is the **maximum** absolute height difference between two consecutive cells on the route. Return the minimum effort needed.

```text
Input:  heights = [[1,2,2],[3,8,2],[5,3,5]]
Output: 2             # route 1,3,5,3,5 has max step 2

Input:  heights = [[1,2,3],[3,8,4],[5,3,5]]
Output: 1             # route 1,2,3,4,5

Input:  heights = [[1,2,1,1,1],[1,2,1,2,1],[1,2,1,2,1],[1,2,1,2,1],[1,1,1,2,1]]
Output: 0
```

**Constraints:** `1 <= rows, cols <= 100`, `1 <= heights[i][j] <= 10^6`.

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Pattern

**Dijkstra with a minimax cost** (Day 34 transferred). A path's cost is `max` over its edges instead of a sum. Dijkstra's greedy invariant still holds because `max` is **monotone**: extending a path never lowers its cost, which plays the same role as non-negative weights. Two equally valid alternatives use other patterns from this week and earlier:

- **Union-Find / Kruskal (Day 32):** sort all edges by difference and union them until the start and the end are connected. The last weight added is the answer.
- **Binary search on the answer (Week 2) + BFS (Day 27):** "can I reach the end using only steps ≤ E?" is monotone in E.

### Intuition

1. `effort[cell]` = the best (smallest) maximum step needed to reach this cell so far.
2. Pop the cell with the smallest effort. For each neighbour, the candidate effort is `max(current effort, abs(height difference))`. If that improves the neighbour, push it.
3. The first time the bottom-right cell is popped (and the pop isn't stale), its effort is final, so return it.

### Complexity

- **Time:** $O(RC \log(RC))$. Each cell has at most 4 edges, so $E = O(RC)$.
- **Space:** $O(RC)$.

### Code

```python
from __future__ import annotations

import heapq
import math


class Solution:
    def minimumEffortPath(self, heights: list[list[int]]) -> int:
        rows, cols = len(heights), len(heights[0])
        effort = [[math.inf] * cols for _ in range(rows)]
        effort[0][0] = 0
        heap: list[tuple[int, int, int]] = [(0, 0, 0)]       # (effort, row, col)

        while heap:
            e, r, c = heapq.heappop(heap)
            if e > effort[r][c]:
                continue                                      # stale entry
            if (r, c) == (rows - 1, cols - 1):
                return e                                      # finalized: minimal possible
            for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                nr, nc = r + dr, c + dc
                if 0 <= nr < rows and 0 <= nc < cols:
                    # Minimax relaxation: path cost is the WORST step on it.
                    candidate = max(e, abs(heights[nr][nc] - heights[r][c]))
                    if candidate < effort[nr][nc]:
                        effort[nr][nc] = candidate
                        heapq.heappush(heap, (candidate, nr, nc))
        raise AssertionError("bottom-right cell is always reachable")


if __name__ == "__main__":
    s = Solution()
    assert s.minimumEffortPath([[1, 2, 2], [3, 8, 2], [5, 3, 5]]) == 2
    assert s.minimumEffortPath([[1, 2, 3], [3, 8, 4], [5, 3, 5]]) == 1
    assert s.minimumEffortPath(
        [[1, 2, 1, 1, 1], [1, 2, 1, 2, 1], [1, 2, 1, 2, 1], [1, 2, 1, 2, 1], [1, 1, 1, 2, 1]]
    ) == 0
    assert s.minimumEffortPath([[7]]) == 0                       # start == end
    assert s.minimumEffortPath([[1, 10, 6, 7, 9, 10, 4, 9]]) == 9  # single row: no choice
    print("all tests passed")
```

**Self-grade:**

- ✅ Solved in 18 min **and** could explain why `max` keeps Dijkstra correct → mastered.
- 🟡 Solved with BFS + binary search or DSU but didn't see the Dijkstra option, or the other way round → fine. Implement the *other* approach on Day 36 as a warm-up.
- 🔴 Not solved → redo LC 743 from scratch, then retry this problem on Day 37.

</details>

---

## E. Self-Audit & Gap Log (5 min)

Rate each item **1–5**: 1 = can't explain, 3 = can explain with notes, 5 = can teach it and spot it in code review.

| Area | Skill | Score |
|---|---|---|
| Imports | Walk through `sys.modules` → `meta_path` → spec → loader → exec | ☐ |
| Imports | Diagnose and fix a circular import three different ways | ☐ |
| Packaging | Explain wheel vs sdist, src layout, ranges vs lockfiles | ☐ |
| ASGI | Write a raw ASGI app and a pure-ASGI middleware from memory | ☐ |
| ASGI | Explain sync vs async endpoints, the threadpool limit, lifespan, and DI caching | ☐ |
| Serialization | Choose a format for a cache, an API and an internal RPC, and justify the choice | ☐ |
| Serialization | List 4 types JSON silently loses, and explain why pickle is RCE | ☐ |
| DSA | Topo sort: Kahn + three-colour DFS; why two colours fail | ☐ |
| DSA | Union-Find with both optimizations; when *not* to use it | ☐ |
| DSA | Dijkstra: stale-entry skip, why non-negative weights matter, its variants | ☐ |
| Spaced | Weeks 1–4 questions S1–S4 answered without peeking | ☐ |

**Action rules:**

- Any **≤ 2** → redo that day's code example from a blank file before Day 36.
- Any **3** → add it to `GAPS.md` with a re-test date during the Day 42 capstone.
- All **≥ 4** → stretch: solve LC 269 (Alien Dictionary) and LC 1584 (Min Cost to Connect All Points, Kruskal).

**Retro (write 3 lines in `GAPS.md`):**

```text
Which of this week's gotchas exists in a codebase I work on right now:
One graph pattern I still confuse with another:
One thing I'll do differently in the final week:
```

---

[← Day 34 — Dijkstra](day-34-dijkstra.md) · [Week 5 overview](README.md) · [Day 36 — Observability →](../week-06/day-36-observability-contextvars-otel.md)
