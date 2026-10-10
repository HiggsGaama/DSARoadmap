# Day 42 — Final Capstone: Consolidation & Self-Audit

[← Week 6](README.md) · **Track:** Audit · **Time box:** 60 min (plus an optional second session for the code review rewrite)

| Block | Minutes | Purpose |
|---|---|---|
| A. Cumulative retrieval quiz | 15 | Six weeks of runtime and service knowledge, recalled cold |
| B. Pattern recognition sprint | 10 | 20 statements, 20 patterns, under 30 s each |
| C. Production code review | 15 | One realistic LLM gateway handler with bugs from every week |
| D. Transfer problem (timed) | 15 | LC 300, with an $O(n^2)$ answer and an $O(n \log n)$ answer |
| E. Final self-audit & next steps | 5 | Decide what the next six weeks look like |

**Rules (same as every consolidation day):** write your answer *before* opening each collapsible. Every miss goes into `GAPS.md`.

---

## A. Cumulative Retrieval Quiz (15 min)

**Q1 (Week 1, memory).** A worker's `tracemalloc` heap is flat but its RSS climbs a little with every request, forever. What category of problem is this, and what tool do you reach for?

<details markdown="1"><summary>Answer</summary>

Python-level objects aren't accumulating, so it's either a **native leak** (C extension, gRPC or numpy buffers) or growing allocator fragmentation. Because RSS keeps climbing rather than plateauing after a spike, suspect a native leak first. Use `memray run --native` (or jemalloc profiling) to find the allocation sites, and recycle workers as a stopgap.

</details>

**Q2 (Week 1, asyncio).** Why can `for k in keys: await cache_get(k)` freeze an event loop even though every call is `async`?

<details markdown="1"><summary>Answer</summary>

`await` gives control back to the loop only if something at the bottom of the chain yields a **pending future**. On cache hits, `cache_get` returns without suspending, so the whole loop runs synchronously. Insert `await asyncio.sleep(0)` periodically, or batch the work.

</details>

**Q3 (Week 2, context managers).** What does it mean when `__exit__` returns `True`, and why is that dangerous in a generic "cleanup" context manager?

<details markdown="1"><summary>Answer</summary>

A truthy return value **suppresses the exception** raised in the `with` body. A cleanup helper that accidentally returns `True` (or returns the result of a call that happens to be truthy) silently swallows errors, including `KeyboardInterrupt` if you aren't careful. Return `None`/`False` unless suppression is the explicit purpose, as with `contextlib.suppress`.

</details>

**Q4 (Week 2, descriptors).** Why can you overwrite a `functools.cached_property` value by assigning to the instance attribute, but not a `property`?

<details markdown="1"><summary>Answer</summary>

Attribute lookup precedence is **data descriptor** (defines `__set__` or `__delete__`) > **instance `__dict__`** > **non-data descriptor**. `property` is a data descriptor, so it always wins. `cached_property` is a non-data descriptor: it writes the computed value into the instance `__dict__`, which shadows it on later lookups. That's also why it's fast after the first access, and why it doesn't work with `__slots__` classes that have no `__dict__`.

</details>

**Q5 (Week 2, GIL).** Your embedding-preprocessing step is pure-Python CPU work. Threads didn't help. What are your options on 3.12 and on 3.13t+?

<details markdown="1"><summary>Answer</summary>

On 3.12, pure-Python bytecode runs serially under the GIL, so use a **process pool** (watch the pickling cost of arguments), move the hot loop into code that releases the GIL (numpy, Rust/C extensions, `tokenizers`), or scale out with more workers. On the **free-threaded build** (3.13t/3.14t), threads can run Python code truly in parallel, at some single-thread overhead, and only if your C extensions support it.

</details>

**Q6 (Week 3, class creation).** You need a plugin registry where every subclass of `Handler` registers itself under a `name`. Metaclass or `__init_subclass__`, and why?

<details markdown="1"><summary>Answer</summary>

Use **`__init_subclass__`**. It runs on every subclass definition with keyword arguments (`class Pdf(Handler, name="pdf")`) and needs no metaclass. Metaclasses conflict when combined (ABCMeta, framework metaclasses) and are harder to reason about. Save them for changing *how classes are built*, for example `__prepare__` ordering or class-level `__getitem__`.

</details>

**Q7 (Week 3, `__slots__`).** What do you gain and lose with `@dataclass(slots=True)` on a class with 10M instances?

<details markdown="1"><summary>Answer</summary>

You gain: no per-instance `__dict__`, which saves a lot of memory (often around 40–60% for small objects), and slightly faster attribute access. You lose: arbitrary attributes, weak references unless you add `__weakref__` (`weakref_slot=True`), `cached_property`, and some multiple-inheritance combinations (layout conflicts).

</details>

**Q8 (Week 3, typing).** What does `isinstance(x, SupportsClose)` check for a `@runtime_checkable Protocol`, and what doesn't it check?

<details markdown="1"><summary>Answer</summary>

It checks only that the required **attributes or methods exist**. It doesn't check their signatures, return types or semantics. Structural typing is enforced statically by the type checker, and the runtime check is a shallow presence test.

</details>

**Q9 (Week 4, cancellation).** Why must code that catches `asyncio.CancelledError` (almost always) re-raise it?

<details markdown="1"><summary>Answer</summary>

Cancellation is how `TaskGroup`, `asyncio.timeout`, server shutdown and client-disconnect handling stop work. Swallowing it makes the task look finished or continue running. The timeout then can't convert the cancellation into `TimeoutError`, shutdown hangs, and sibling tasks aren't cleaned up. Do your cleanup, then `raise`. Also note that `except Exception` doesn't catch it (it's a `BaseException` since 3.8), but `except BaseException` or a bare `except:` does.

</details>

**Q10 (Week 4, micro-batching).** Your inference micro-batcher has `max_batch=32, max_wait=50 ms`. At low traffic, p50 latency is about 50 ms higher than the model's latency. Why, and what do you change?

<details markdown="1"><summary>Answer</summary>

At low traffic a batch never fills, so every request waits the full `max_wait` timer. Lower `max_wait`, or make it adaptive: dispatch immediately when the GPU is idle and batch only while a batch is in flight. Batch size and wait time trade throughput against tail latency, so tune them against your SLO.

</details>

**Q11 (Week 5, ASGI/FastAPI).** Why is an `async def` endpoint that calls a sync SDK worse than a plain `def` endpoint calling the same SDK?

<details markdown="1"><summary>Answer</summary>

FastAPI/Starlette runs plain `def` endpoints in a **threadpool**, so the event loop stays free. An `async def` endpoint runs **on the event loop**, so the sync call blocks every concurrent request in that worker (Day 5).

</details>

**Q12 (Week 5, serialization).** A teammate proposes caching model responses as `pickle` blobs in Redis, which is shared with other teams. What's the problem?

<details markdown="1"><summary>Answer</summary>

**Unpickling runs arbitrary code** (`__reduce__`). Anyone who can write to that Redis can get remote code execution in your service. Use a data-only format (JSON, msgpack, protobuf) with explicit schemas. Pickle also ties the cache to class definitions and Python versions.

</details>

**Q13 (Week 6, observability).** Logs written inside `loop.run_in_executor(None, parse_pdf, ...)` have `request_id="-"`. Explain the cause and give two fixes.

<details markdown="1"><summary>Answer</summary>

`run_in_executor` doesn't copy the `contextvars` context into the worker thread. Fix it with `await asyncio.to_thread(parse_pdf, ...)`, which copies the context, or with `loop.run_in_executor(pool, contextvars.copy_context().run, parse_pdf, ...)`.

</details>

**Q14 (Week 6, resilience + streaming).** (a) Why does a half-open circuit breaker allow exactly one probe? (b) Streaming works locally but users get the full answer at once in production. What's the first thing you check?

<details markdown="1"><summary>Answer</summary>

(a) If a recovering dependency suddenly gets *all* the waiting traffic, it falls over again. One probe tests health without stampeding it. (b) **Response buffering** somewhere on the path: nginx `proxy_buffering` (send `X-Accel-Buffering: no`), compression middleware, or a CDN or API gateway. Also confirm heartbeats are more frequent than the idle timeouts on the path.

</details>

---

## B. Pattern Recognition Sprint (10 min)

Name the pattern and the phrase that gave it away.

1. "For each day, how many days until a warmer temperature?"
2. "Minimum eating speed so all banana piles are finished within `h` hours."
3. "Return the node where a linked list's cycle begins, using O(1) memory."
4. "Length of the longest path between any two nodes in a binary tree."
5. "Return the values visible when looking at a binary tree from the right."
6. "Return the k-th smallest value in a BST."
7. "Return the k closest points to the origin from a stream of 10^7 points."
8. "Minimum number of meeting rooms required for a list of meetings."
9. "Count islands of `1`s in a grid (4-directional)."
10. "Given package dependencies, return a valid build order or report a cycle."
11. "Merge accounts that share any email address."
12. "Lowest-latency route between two regions; all link latencies are positive."
13. "Shortest number of moves from S to E in a maze grid, every move costs 1."
14. "Max money robbing houses in a row without robbing two adjacent ones."
15. "Minimum insert/delete/replace operations to turn `word1` into `word2`."
16. "Generate every valid string of `n` pairs of parentheses."
17. "Smallest substring of `s` containing every character of `t`."
18. "In a sorted array, find two numbers whose sum is closest to `target`."
19. "Detect whether any two requests in an unsorted log share a fingerprint, in O(n)."
20. "Find every dictionary word that can be traced through adjacent cells of a letter grid."

<details markdown="1"><summary>Answers</summary>

| # | Pattern | Giveaway |
|---|---|---|
| 1 | Monotonic stack (decreasing) | "next greater / warmer" |
| 2 | Binary search on the answer | "minimum speed such that…"; feasibility is monotone in speed |
| 3 | Fast/slow pointers (Floyd), then reset one pointer to the head | "cycle", "O(1) memory" |
| 4 | Tree DFS, post-order returning height | "any two nodes": combine left and right depths at each node |
| 5 | Tree BFS, level order (last node per level) | "from the right": per-level view |
| 6 | BST in-order traversal (sorted order), stop at k | "BST" + "k-th smallest" |
| 7 | Heap of size k (max-heap on distance) | "k closest", "stream": O(n log k), O(k) memory |
| 8 | Intervals: sort by start + min-heap of end times (or a sweep line) | "meetings", "rooms" |
| 9 | Grid DFS/BFS flood fill | "islands", "grid" |
| 10 | Topological sort (Kahn's algorithm; leftover nodes mean a cycle) | "dependencies", "order", "cycle" |
| 11 | Union-Find | "merge", "share any" (connectivity under unions) |
| 12 | Dijkstra | "lowest total", "positive weights" |
| 13 | **Plain BFS**, not Dijkstra | "every move costs 1" (unweighted) |
| 14 | 1-D DP (`dp[i] = max(dp[i-1], dp[i-2] + v)`) | "max", "no two adjacent" |
| 15 | 2-D DP (edit distance) | two strings, "minimum operations" |
| 16 | Backtracking with pruning (open/close counts) | "generate every valid" |
| 17 | Sliding window, variable size, shortest-valid flip | "smallest substring containing" |
| 18 | Two pointers, converging | "sorted", "two numbers", "closest" |
| 19 | Hashing: membership set | "unsorted", "O(n)", "share" |
| 20 | Backtracking + trie (LC 212) | "every dictionary word", "grid": prefix pruning |

**Score:** 18+ means your pattern recognition is solid. Under 15: rebuild a one-page cheat sheet of *giveaway phrase → pattern* from every week's §2 and re-test in one week.

</details>

---

## C. Production Code Review (15 min)

This FastAPI LLM gateway endpoint passed code review and load-tested fine at 5 RPS. At 200 RPS in production you see: latency spikes across *all* routes, logs attributing errors to the wrong customers, one tenant receiving another tenant's cached completion, memory growth, streams arriving all at once, and a provider outage turning into a full gateway outage. **Find at least 12 defects.** Then, ideally in a second session, write the fixed version.

```python
import asyncio
import logging
import threading
import time

import httpx
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

app = FastAPI()
log = logging.getLogger("gateway")
_local = threading.local()
_response_cache: dict[str, str] = {}
client = httpx.AsyncClient(base_url="https://llm.internal")


def count_tokens(text: str) -> int:
    return len(tokenizer.encode(text))          # ~40 ms of CPU for long prompts


async def record_usage(request_id: str, tokens: int) -> None:
    ...


async def call_with_retry(payload: dict) -> httpx.Response:
    for attempt in range(5):
        try:
            resp = await client.post("/v1/generate", json=payload)
            resp.raise_for_status()
            return resp
        except BaseException:
            time.sleep(2 ** attempt)
    raise RuntimeError("upstream failed")


async def stream_tokens(payload: dict):
    async with client.stream("POST", "/v1/generate", json=payload) as resp:
        async for line in resp.aiter_lines():
            yield f"data: {line}\n"


@app.post("/v1/chat")
async def chat(request: Request):
    body = await request.json()
    _local.request_id = request.headers.get("x-request-id")
    log.info(f"chat request {_local.request_id} body={body}")
    tokens = count_tokens(body["prompt"])

    if body.get("stream"):
        return StreamingResponse(stream_tokens(body), media_type="text/event-stream")

    if body["prompt"] in _response_cache:
        return {"text": _response_cache[body["prompt"]]}

    resp = await call_with_retry(body)
    _response_cache[body["prompt"]] = resp.json()["text"]
    asyncio.create_task(record_usage(_local.request_id, tokens))
    return resp.json()
```

<details markdown="1"><summary>Defects (16) & fixed version</summary>

| # | Defect | Production symptom | Where you learned it |
|---|---|---|---|
| 1 | `threading.local` for the request ID | Concurrent requests overwrite each other's ID, so logs blame the wrong customer | Day 36 |
| 2 | f-string log of the full `body` | Eager formatting on every request; **prompts/PII/keys in logs** | Day 36 |
| 3 | `count_tokens` (CPU) runs on the event loop | 40 ms stall per request × 200 RPS: every route's latency spikes | Day 5, Week 2 (GIL) |
| 4 | `except BaseException` in the retry loop | Swallows `CancelledError`: timeouts and shutdown break, and abandoned requests keep retrying | Week 4 |
| 5 | Retries **everything**, including 4xx | Bad requests and auth errors are retried 5×, burning quota | Day 38 |
| 6 | `time.sleep` inside `async def` | Up to 1+2+4+8+16 s of **whole-loop** freeze per failing request | Day 5 |
| 7 | No jitter, no deadline, no budget, no breaker | A provider brownout becomes a synchronized retry storm and a pool exhaustion | Day 38 |
| 8 | Global, unbounded `_response_cache` | Memory grows forever | Day 1 |
| 9 | Cache key = prompt only, shared across tenants and params | **Cross-tenant data leak**, and wrong answers when model/temperature differ | Security + Day 1 |
| 10 | SSE frames end with `\n`, not `\n\n` | Clients never see an event boundary, so nothing renders until the stream ends | Day 40 |
| 11 | No `X-Accel-Buffering: no` / `Cache-Control: no-cache` | nginx buffers the stream: "streaming" arrives all at once | Day 40 |
| 12 | Streaming path skips usage accounting and rate limiting | Unmetered spend, and provider 429s under load | Day 38 |
| 13 | `asyncio.create_task` result discarded | Usage records can be GC'd mid-flight or lost on shutdown | Day 5 |
| 14 | Module-level `AsyncClient` created at import, never closed, no explicit timeouts | Not bound to the app lifespan; leaked connections on reload; implicit default timeouts | Week 5 (ASGI lifespan) |
| 15 | No request model: `body["prompt"]` on raw JSON | Missing field → `KeyError` → 500 instead of 422; no size limits | Week 3 (Pydantic) |
| 16 | `resp.json()` parsed twice; `_local.request_id` may be `None` | Wasted CPU on large responses; usage rows with a null ID | Week 5 (serialization) |

**Fixed version (sketch, needs `fastapi` + `httpx`; resilience helpers from Day 38):**

```python
from __future__ import annotations

import asyncio
import contextvars
import hashlib
import json
import logging
import uuid
from collections.abc import AsyncIterator
from contextlib import aclosing, asynccontextmanager

import httpx
from cachetools import TTLCache
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field

log = logging.getLogger("gateway")
request_id: contextvars.ContextVar[str] = contextvars.ContextVar("request_id", default="-")
_cache: TTLCache[str, str] = TTLCache(maxsize=10_000, ttl=300)       # bounded + expiring
_background: set[asyncio.Task[None]] = set()


class ChatRequest(BaseModel):
    tenant_id: str
    model: str
    prompt: str = Field(max_length=200_000)
    temperature: float = 0.0
    stream: bool = False


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    app.state.client = httpx.AsyncClient(
        base_url="https://llm.internal",
        timeout=httpx.Timeout(connect=2.0, read=30.0, write=5.0, pool=2.0),
    )
    yield
    await asyncio.gather(*_background, return_exceptions=True)        # drain usage records
    await app.state.client.aclose()


app = FastAPI(lifespan=lifespan)


def cache_key(req: ChatRequest) -> str:
    raw = json.dumps([req.tenant_id, req.model, req.temperature, req.prompt])
    return hashlib.sha256(raw.encode()).hexdigest()                  # tenant-scoped, param-aware


def spawn(coro) -> None:
    task = asyncio.create_task(coro)
    _background.add(task)
    task.add_done_callback(_background.discard)


async def sse_frames(client: httpx.AsyncClient, req: ChatRequest) -> AsyncIterator[bytes]:
    async with client.stream("POST", "/v1/generate", json=req.model_dump()) as resp:
        resp.raise_for_status()
        async with aclosing(resp.aiter_lines()) as lines:            # upstream closed on disconnect
            async for line in lines:
                if line:
                    yield f"data: {line}\n\n".encode()                # blank-line terminator
    yield b"event: end\ndata: [DONE]\n\n"


@app.post("/v1/chat")
async def chat(req: ChatRequest, request: Request):
    token = request_id.set(request.headers.get("x-request-id") or uuid.uuid4().hex)
    try:
        log.info("chat request", extra={"tenant": req.tenant_id, "model": req.model,
                                        "prompt_chars": len(req.prompt)})          # no raw prompt
        tokens = await asyncio.to_thread(count_tokens, req.prompt)                 # off the loop
        await rate_limiter.acquire()                                               # Day 38 bucket
        client: httpx.AsyncClient = request.app.state.client

        if req.stream:
            spawn(record_usage(request_id.get(), tokens))
            return StreamingResponse(
                sse_frames(client, req),
                media_type="text/event-stream",
                headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
            )

        key = cache_key(req)
        if (hit := _cache.get(key)) is not None:
            return {"text": hit, "cached": True}

        data = await breaker.call(lambda: retry_call(                              # Day 38
            lambda: post_generate(client, req), policy=POLICY, budget=BUDGET,
            deadline=asyncio.get_running_loop().time() + 20, idempotent=True, rng=RNG))
        _cache[key] = data["text"]
        spawn(record_usage(request_id.get(), tokens))
        return data
    finally:
        request_id.reset(token)
```

The sketch uses `count_tokens`, `record_usage`, `post_generate` (one POST that raises `RetryableError` / `NonRetryableError` based on status and returns parsed JSON), `rate_limiter`, `breaker`, `retry_call` and the policy objects from earlier days. Wiring them together is the stretch exercise.

</details>

---

## D. Transfer Problem — LeetCode 300: Longest Increasing Subsequence (Medium) · 15 min timed

No hints. Identify the state yourself.

**Statement:** Given an integer array `nums`, return the length of the longest **strictly increasing subsequence**.

```text
Input:  nums = [10, 9, 2, 5, 3, 7, 101, 18]
Output: 4            # [2, 3, 7, 101] (or [2, 5, 7, 18], ...)

Input:  nums = [0, 1, 0, 3, 2, 3]
Output: 4

Input:  nums = [7, 7, 7, 7]
Output: 1
```

**Constraints:** `1 <= len(nums) <= 2500`, `-10^4 <= nums[i] <= 10^4`. Follow-up: can you do it in $O(n \log n)$?

<details markdown="1">
<summary><strong>Solution & Explanation</strong></summary>

### Approach 1: 1-D DP, "ending at i" (Day 37 family)

- **State:** `dp[i]` = length of the longest increasing subsequence that **ends exactly at index `i`**.
- **Transition:** `dp[i] = 1 + max(dp[j] for j < i if nums[j] < nums[i])`, or `1` if there's no such `j`.
- **Answer:** `max(dp)`, not `dp[-1]`, because the best subsequence can end anywhere.
- **Complexity:** $O(n^2)$ time, $O(n)$ space. With `n = 2500` that's about 3M steps, which is fine.

### Approach 2: patience sorting + binary search (Week 2 pattern)

Maintain `tails`, where `tails[k]` = the **smallest possible tail value** of any increasing subsequence of length `k + 1` seen so far.

- `tails` is always strictly increasing. (If a length-3 subsequence ended at 5, then a length-2 subsequence ends somewhere below 5.)
- For each `x`, find the first `tails[k] >= x` with `bisect_left`:
  - If there isn't one, `x` extends the longest subsequence: append it.
  - Otherwise, replace `tails[k] = x`. A smaller tail for length `k + 1` can only help future elements extend it.
- **Answer:** `len(tails)`. Note that `tails` itself is **not** necessarily a valid LIS. It's a summary of the best tail per length.
- `bisect_left` (not `bisect_right`) enforces **strictly** increasing: an equal value replaces instead of extending.
- **Complexity:** $O(n \log n)$ time, $O(n)$ space.

### Code

```python
from __future__ import annotations

import random
from bisect import bisect_left


class Solution:
    def lengthOfLIS(self, nums: list[int]) -> int:
        """O(n log n): tails[k] = smallest tail of an increasing subsequence of length k+1."""
        tails: list[int] = []
        for x in nums:
            k = bisect_left(tails, x)    # first tail >= x (strictly increasing => left)
            if k == len(tails):
                tails.append(x)          # x extends the longest subsequence so far
            else:
                tails[k] = x             # better (smaller) tail for length k+1
        return len(tails)


def lis_quadratic(nums: list[int]) -> int:
    """O(n^2) DP: dp[i] = LIS length ending exactly at i."""
    dp = [1] * len(nums)
    for i in range(len(nums)):
        for j in range(i):
            if nums[j] < nums[i] and dp[j] + 1 > dp[i]:
                dp[i] = dp[j] + 1
    return max(dp)


if __name__ == "__main__":
    s = Solution()
    assert s.lengthOfLIS([10, 9, 2, 5, 3, 7, 101, 18]) == 4
    assert s.lengthOfLIS([0, 1, 0, 3, 2, 3]) == 4
    assert s.lengthOfLIS([7, 7, 7, 7]) == 1
    assert s.lengthOfLIS([5]) == 1
    assert s.lengthOfLIS(list(range(2500))) == 2500
    assert s.lengthOfLIS(list(range(2500, 0, -1))) == 1

    rng = random.Random(42)  # cross-check both approaches on random inputs
    for _ in range(300):
        arr = [rng.randint(-20, 20) for _ in range(rng.randint(1, 40))]
        assert s.lengthOfLIS(arr) == lis_quadratic(arr), arr
    print("all tests passed")
```

### Dry run of `tails`: `[10, 9, 2, 5, 3, 7, 101, 18]`

| `x` | `bisect_left` | Action | `tails` |
|---|---|---|---|
| 10 | 0 (empty) | append | [10] |
| 9 | 0 | replace | [9] |
| 2 | 0 | replace | [2] |
| 5 | 1 | append | [2, 5] |
| 3 | 1 | replace | [2, 3] |
| 7 | 2 | append | [2, 3, 7] |
| 101 | 3 | append | [2, 3, 7, 101] |
| 18 | 3 | replace | [2, 3, 7, 18] → **length 4** |

**Self-grade:**

- ✅ Got the $O(n^2)$ DP in 15 min **and** can explain why `tails` stays sorted → DP and binary search are both solid.
- 🟡 Got only $O(n^2)$ → fine for interviews. Revisit "binary search on a maintained invariant" from Week 2.
- 🔴 Neither → redo Day 37 (state / transition / base / order), then LC 300 again in three days.

</details>

---

## E. Final Self-Audit & What's Next (5 min)

Rate each item 1–5: 1 = can't explain, 3 = can explain with notes, 5 = can teach it and catch it in code review.

| Week | Area | Skill | Score |
|---|---|---|---|
| 1 | Runtime | Refcount vs cyclic GC; leak vs fragmentation; generator and event-loop mechanics | ☐ |
| 2 | Runtime | Context managers and exceptions; descriptor precedence; GIL and parallelism options | ☐ |
| 3 | Object model | Class creation, `__init_subclass__`; slots/dataclass/Pydantic trade-offs; Protocols and generics | ☐ |
| 4 | Concurrency | Cancellation rules; backpressure and micro-batching; profiling with py-spy and memray | ☐ |
| 5 | Plumbing | Import system and packaging; ASGI lifecycle; safe, fast serialization | ☐ |
| 6 | Services | contextvars + structured logs + traces; retries/breakers/rate limits; streaming with disconnect cleanup | ☐ |
| 1–2 | DSA | Hashing, two pointers, sliding window, monotonic stack, binary search, fast/slow | ☐ |
| 3–4 | DSA | Tree DFS/BFS, BST, heaps, intervals, grid traversal | ☐ |
| 5–6 | DSA | Topological sort, union-find, Dijkstra, 1-D/2-D DP, backtracking and tries | ☐ |

**Action rules:** any area ≤ 2 → re-run that week's consolidation day next week. Any area at 3 → add its days to `GAPS.md` and re-test in 2 weeks. All ≥ 4 → you're ready for the next phase.

### What next after 6 weeks

1. **Build the capstone (2–3 weeks of evenings):** a small **LLM gateway** that does everything this plan covered: FastAPI with a lifespan-managed `httpx` client, Pydantic request models, contextvars request IDs + JSON logs + OpenTelemetry traces, retry/breaker/token-bucket around a mock provider, SSE streaming with heartbeats and disconnect cleanup, micro-batched embeddings, and a load test (`locust` or `k6`) whose results you profile with `py-spy` and `memray`. Write a one-page design doc. It becomes your strongest interview story.
2. **DSA maintenance (3 × 30 min per week):** work through the remaining NeetCode 150 problems by pattern, and re-solve every problem in your gap log after 1, 7 and 30 days (spaced repetition beats volume).
3. **System design for AI platforms:** read *Designing Data-Intensive Applications* (Kleppmann), chapters 1–9, and the *Google SRE Book* chapters on overload and cascading failures. Then design, on paper: a multi-tenant rate-limited LLM gateway, a RAG ingestion pipeline with backpressure, and a GPU inference autoscaler.
4. **Go deeper on one runtime topic:** read CPython's `Objects/genobject.c` and `Lib/asyncio/tasks.py` (the `__step` method from Day 5), or contribute a small fix to an async library you use.

---

[← Day 41 — Backtracking & Tries](day-41-backtracking-tries.md) · [Week 6 overview](README.md) · [Curriculum home →](../README.md)
