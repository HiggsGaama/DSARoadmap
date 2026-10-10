# Day 31 — ASGI & FastAPI Internals: What Actually Happens to a Request

[← Week 5](README.md) · **Track:** Python Systems · **Time box:** 60 min (concept 20 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** Model gateways, embedding services and RAG backends run on FastAPI/Starlette over uvicorn or hypercorn. Once you see that the whole stack is a single `async def app(scope, receive, send)` callable, a lot becomes debuggable: middleware ordering, streaming that buffers anyway, startup that loads the model 8 times, endpoints that stall under load.

**Prerequisites:** Day 5 (event loop, `to_thread`), Day 29 (import-time side effects).

---

## 1. Core Concept

### 1.1 The ASGI contract: one callable, three arguments

```python
async def app(scope: dict, receive: Callable[[], Awaitable[dict]], send: Callable[[dict], Awaitable[None]]) -> None: ...
```

| Piece | What it is | HTTP example |
|---|---|---|
| `scope` | Connection metadata, a plain dict, read once | `{"type": "http", "method": "POST", "path": "/embed", "headers": [(b"content-type", b"application/json")], "query_string": b"", "client": (...), "state": {...}}` |
| `receive()` | Awaits the next **event from the client** | `{"type": "http.request", "body": b"...", "more_body": True}` … then `{"type": "http.disconnect"}` |
| `send(msg)` | Emits an **event to the client** | `{"type": "http.response.start", "status": 200, "headers": [...]}` then one or more `{"type": "http.response.body", "body": b"...", "more_body": bool}` |

The server (uvicorn, hypercorn, granian) parses bytes off the socket, builds `scope`, and calls your app **once per connection/request as a coroutine on its event loop**. Everything else, including routing, request objects, validation and dependency injection, is library code layered on these three primitives.

**Things the contract makes explicit:**

- **Bodies arrive in chunks.** A large upload is many `http.request` messages, and only the last one has `more_body: False`. A handler that reads a single message silently truncates the body.
- **Headers are a list of byte pairs,** not a dict. Duplicates (`set-cookie`) are legal, and names arrive lowercased.
- **Streaming** means sending several `http.response.body` messages with `more_body: True`. Nothing about the protocol buffers. If your stream arrives all at once, something in the middle (middleware, a proxy such as nginx with `proxy_buffering on`, or gzip) is buffering it.
- **Lifespan** is a separate scope type, `{"type": "lifespan"}`, run **once per worker process**. The server sends `lifespan.startup`, and the app replies `lifespan.startup.complete` (or `.failed`). This is where model loading, connection pools and HTTP clients belong, *not* at import time (Day 29). The `state` dict set during startup is shallow-copied into every request's `scope["state"]`.
- **WebSockets** use `scope["type"] == "websocket"` with `websocket.connect` / `.receive` / `.send` events, the same shape.

### 1.2 How Starlette and FastAPI layer on top

```text
uvicorn  ─calls─►  FastAPI.__call__(scope, receive, send)
                      └─ middleware stack (outermost first: ServerErrorMiddleware → user middleware → ExceptionMiddleware)
                           └─ Router: match scope["path"] + method against routes
                                └─ route handler: build Request(scope, receive), resolve dependencies,
                                   validate (Pydantic), call your endpoint, serialize, send()
```

- **Middleware is just ASGI wrapping ASGI:** `class M: def __init__(self, app): ...; async def __call__(self, scope, receive, send)`. It can inspect or modify `scope`, wrap `receive` (to see or alter the request body), and wrap `send` (to add headers or measure timing). `app.add_middleware()` inserts *outside* the previously added ones, so **the last one added runs first**.
- **`async def` endpoints run directly on the event loop.** Any blocking call inside them stalls every request in the worker (Day 5).
- **Plain `def` endpoints, and `def` dependencies, run in a threadpool** via `anyio.to_thread.run_sync`. AnyIO's default limiter allows **40 concurrent threads per process**.
- **Dependency injection:** FastAPI inspects your function signature once at startup and builds a dependency tree. Per request, it resolves the dependencies, **caching each one within the request** by default (`Depends(get_db)` used twice yields one session). Dependencies that use `yield` act like context managers: the code before `yield` runs before the endpoint, and the teardown runs after the response, so don't do slow work in the teardown.
- **Response serialization:** the return value is validated against `response_model` (with Pydantic v2's Rust core), `jsonable_encoder`'d where needed, and JSON-encoded. Returning a `Response` subclass directly (`ORJSONResponse`, `StreamingResponse`) skips parts of this pipeline (Day 33).

---

## 2. Production Gotchas

### 2a. Sync endpoints and the 40-thread ceiling

```python
@app.get("/report")
def report() -> dict:                    # sync def -> runs in the AnyIO threadpool
    rows = legacy_db.query(BIG_SQL)      # 2 s blocking call
    return summarize(rows)
```

That's correct, since it doesn't block the loop, but every in-flight request holds one of **40** threadpool tokens. The 41st concurrent request queues, *invisibly*, before your endpoint even starts. Latency climbs in steps while CPU stays idle. The same pool also runs every sync `Depends()` and file response, so a slow sync endpoint can starve unrelated routes.

**Fix:** use async drivers for hot paths. Size the limiter explicitly at startup if you must keep sync code (`anyio.to_thread.current_default_thread_limiter().total_tokens = N`) and load-test with that setting. Also export threadpool queue depth as a metric. Never write `async def` and then call sync I/O inside, which is strictly worse: it blocks the loop itself.

### 2b. Middleware that consumes the body, or breaks streaming

```python
class AuditBody:
    def __init__(self, app): self.app = app
    async def __call__(self, scope, receive, send):
        if scope["type"] == "http":
            msg = await receive()                 # 🚨 consumes the first body chunk
            log_body(msg.get("body", b""))
        await self.app(scope, receive, send)      # endpoint now waits for a body that never comes
```

`receive` is a **one-shot stream**. If middleware reads from it, it must **replay** the messages to the inner app through a wrapped `receive`. Otherwise the endpoint hangs until the client times out, or receives a truncated body. The response side has the same issue: middleware that collects the body to modify it (compression, response logging) turns streaming LLM output into one big buffered response.

`BaseHTTPMiddleware` (the `@app.middleware("http")` decorator) has a history of issues in exactly this area: streaming, background tasks, and `contextvars` propagation. Starlette's own docs suggest **pure ASGI middleware** (as in the code below) for anything that touches streaming bodies or must be cheap.

---

## 3. Runnable Code Example

It's a raw ASGI app with lifespan, chunked-body parsing, a thread-offloaded "sync endpoint" and a streaming route, wrapped in a streaming-safe timing middleware. A tiny in-process harness plays the server, so you don't need uvicorn. Save as `day31_asgi.py`.

```python
"""
Day 31 — ASGI from first principles: app + pure-ASGI middleware + lifespan,
chunked request bodies, threadpool offload, and streaming, driven by an
in-process "server" harness (no uvicorn required).

Run:  python day31_asgi.py        (Python 3.12+)
"""
from __future__ import annotations

import asyncio
import json
import time
from collections.abc import Awaitable, Callable, MutableMapping, Sequence
from typing import Any

type Scope = MutableMapping[str, Any]
type Message = MutableMapping[str, Any]
type Receive = Callable[[], Awaitable[Message]]
type Send = Callable[[Message], Awaitable[None]]
type ASGIApp = Callable[[Scope, Receive, Send], Awaitable[None]]


# ---------------------------------------------------------------------------
# The application
# ---------------------------------------------------------------------------
async def read_body(receive: Receive) -> bytes:
    """Bodies arrive in chunks: keep reading until more_body is False."""
    chunks: list[bytes] = []
    while True:
        message = await receive()
        if message["type"] == "http.disconnect":
            raise ConnectionAbortedError("client disconnected mid-body")
        chunks.append(message.get("body", b""))
        if not message.get("more_body", False):
            return b"".join(chunks)


async def send_json(send: Send, status: int, payload: Any) -> None:
    body = json.dumps(payload).encode()
    await send({
        "type": "http.response.start",
        "status": status,
        "headers": [(b"content-type", b"application/json"),
                    (b"content-length", str(len(body)).encode())],
    })
    await send({"type": "http.response.body", "body": body})


def blocking_price_lookup(sku: str) -> dict[str, Any]:
    """Stands in for a sync DB driver / SDK call."""
    time.sleep(0.1)
    return {"sku": sku, "price": 9.99}


async def app(scope: Scope, receive: Receive, send: Send) -> None:
    if scope["type"] == "lifespan":
        while True:
            message = await receive()
            if message["type"] == "lifespan.startup":
                scope["state"]["model"] = "embedder-v3 (loaded once per worker)"
                await send({"type": "lifespan.startup.complete"})
            elif message["type"] == "lifespan.shutdown":
                await send({"type": "lifespan.shutdown.complete"})
                return

    assert scope["type"] == "http"
    method, path = scope["method"], scope["path"]

    if method == "POST" and path == "/embed":
        payload = json.loads(await read_body(receive))
        await send_json(send, 200, {"model": scope["state"]["model"], "chars": len(payload["text"])})
    elif method == "GET" and path.startswith("/price/"):
        # What FastAPI does for a plain `def` endpoint: run it in a worker thread.
        result = await asyncio.to_thread(blocking_price_lookup, path.removeprefix("/price/"))
        await send_json(send, 200, result)
    elif method == "GET" and path == "/stream":
        await send({"type": "http.response.start", "status": 200,
                    "headers": [(b"content-type", b"text/plain")]})
        for i in range(3):
            await send({"type": "http.response.body", "body": f"token-{i} ".encode(), "more_body": True})
            await asyncio.sleep(0.01)                      # model producing the next token
        await send({"type": "http.response.body", "body": b"", "more_body": False})
    else:
        await send_json(send, 404, {"detail": "not found"})


# ---------------------------------------------------------------------------
# Pure ASGI middleware: wraps `send`, never buffers the body -> streaming-safe
# ---------------------------------------------------------------------------
def timing_middleware(inner: ASGIApp) -> ASGIApp:
    async def middleware(scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":
            await inner(scope, receive, send)              # pass lifespan/websocket through untouched
            return
        start = time.perf_counter()

        async def send_with_timing(message: Message) -> None:
            if message["type"] == "http.response.start":
                elapsed_ms = (time.perf_counter() - start) * 1000
                headers = [*message.get("headers", []), (b"x-process-ms", f"{elapsed_ms:.1f}".encode())]
                message = {**message, "headers": headers}
            await send(message)

        await inner(scope, receive, send_with_timing)

    return middleware


# ---------------------------------------------------------------------------
# A toy "server": builds scopes, feeds receive(), collects send()
# ---------------------------------------------------------------------------
class Lifespan:
    def __init__(self, asgi_app: ASGIApp) -> None:
        self.app = asgi_app
        self.state: dict[str, Any] = {}
        self._inbox: asyncio.Queue[Message] = asyncio.Queue()
        self._outbox: asyncio.Queue[Message] = asyncio.Queue()
        self._task: asyncio.Task[None] | None = None

    async def __aenter__(self) -> dict[str, Any]:
        scope: Scope = {"type": "lifespan", "asgi": {"version": "3.0"}, "state": self.state}
        self._task = asyncio.create_task(self.app(scope, self._inbox.get, self._outbox.put))
        await self._inbox.put({"type": "lifespan.startup"})
        reply = await self._outbox.get()
        assert reply["type"] == "lifespan.startup.complete", reply
        return self.state

    async def __aexit__(self, *exc_info: object) -> None:
        await self._inbox.put({"type": "lifespan.shutdown"})
        reply = await self._outbox.get()
        assert reply["type"] == "lifespan.shutdown.complete", reply
        assert self._task is not None
        await self._task


async def request(
    asgi_app: ASGIApp, method: str, path: str, *,
    body_chunks: Sequence[bytes] = (b"",), state: dict[str, Any] | None = None,
) -> tuple[int, dict[str, str], bytes, int]:
    scope: Scope = {
        "type": "http", "asgi": {"version": "3.0"}, "http_version": "1.1",
        "method": method, "path": path, "query_string": b"", "headers": [],
        "state": dict(state or {}),                        # shallow copy per request, like real servers
    }
    pending: list[Message] = [
        {"type": "http.request", "body": chunk, "more_body": i < len(body_chunks) - 1}
        for i, chunk in enumerate(body_chunks)
    ]
    sent: list[Message] = []

    async def receive() -> Message:
        return pending.pop(0) if pending else {"type": "http.disconnect"}

    async def send(message: Message) -> None:
        sent.append(message)

    await asgi_app(scope, receive, send)
    start = next(m for m in sent if m["type"] == "http.response.start")
    headers = {k.decode(): v.decode() for k, v in start.get("headers", [])}
    body_messages = [m for m in sent if m["type"] == "http.response.body"]
    body = b"".join(m.get("body", b"") for m in body_messages)
    return start["status"], headers, body, len(body_messages)


async def main() -> None:
    wrapped = timing_middleware(app)
    async with Lifespan(wrapped) as state:
        print("startup state:", state)

        # 1. Body split across 3 chunks, like a large upload over a slow link.
        status, headers, body, _ = await request(
            wrapped, "POST", "/embed",
            body_chunks=[b'{"text": "retrieval-aug', b'mented gene', b'ration"}'], state=state)
        print(f"\nPOST /embed -> {status} {body.decode()}  x-process-ms={headers['x-process-ms']}")

        # 2. Five concurrent "sync endpoints": threads overlap, so ~0.1 s total, not 0.5 s.
        t0 = time.perf_counter()
        results = await asyncio.gather(*(request(wrapped, "GET", f"/price/sku-{i}") for i in range(5)))
        print(f"\n5 x GET /price/* -> {[r[0] for r in results]} in {time.perf_counter() - t0:.2f}s")

        # 3. Streaming: several body messages reach the client separately.
        status, _, body, n_messages = await request(wrapped, "GET", "/stream")
        print(f"\nGET /stream -> {status} {body.decode()!r} in {n_messages} body messages")

        status, _, body, _ = await request(wrapped, "GET", "/nope")
        print(f"\nGET /nope -> {status} {body.decode()}")
    print("\nshutdown complete")


if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output (timings vary):**

```text
startup state: {'model': 'embedder-v3 (loaded once per worker)'}

POST /embed -> 200 {"model": "embedder-v3 (loaded once per worker)", "chars": 30}  x-process-ms=0.1

5 x GET /price/* -> [200, 200, 200, 200, 200] in 0.11s

GET /stream -> 200 'token-0 token-1 token-2 ' in 4 body messages

GET /nope -> 404 {"detail": "not found"}

shutdown complete
```

### The same ideas in FastAPI (illustrative, not part of the runnable file)

```python
from contextlib import asynccontextmanager

import httpx
from fastapi import Depends, FastAPI, Header
from fastapi.responses import StreamingResponse


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(timeout=5.0)    # one pooled client per worker
    app.state.model = load_model()                     # once per worker, not at import
    yield
    await app.state.http.aclose()


app = FastAPI(lifespan=lifespan)


async def get_tenant(x_tenant: str = Header()) -> str:  # dependency; cached per request
    return x_tenant


@app.post("/embed")
async def embed(req: EmbedRequest, tenant: str = Depends(get_tenant)) -> EmbedResponse:
    ...                                                 # async def: must never block


@app.get("/report")
def report() -> Report:                                 # sync def: runs in the 40-token threadpool
    ...


@app.get("/stream")
async def stream() -> StreamingResponse:
    async def tokens():
        for tok in generate():
            yield tok
    return StreamingResponse(tokens(), media_type="text/plain")
```

**Exercises (10 min):**

1. Replace `read_body` with a single `await receive()` and rerun. Read the `JSONDecodeError`: that's the truncated-body bug.
2. Write a `body_logging_middleware` that reads every request chunk, logs the total size, and **replays** the chunks to the inner app through a wrapped `receive`. Verify `/embed` still works.
3. Change `/price` to call `blocking_price_lookup` directly, without `to_thread`. Rerun and compare the time for 5 concurrent requests.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `ASGI explained python scope receive send` | **PyCon / EuroPython** talks (e.g. by Tom Christie, the author of Starlette/uvicorn) | The protocol from the people who designed the ecosystem |
| Deep dive | `FastAPI internals dependency injection how it works` | **ArjanCodes**, conference talks by Sebastián Ramírez (FastAPI author) | How signatures become a dependency graph; sync vs async endpoint execution |
| Practical | `anthonywritescode asgi` / `mCoding fastapi` | **anthonywritescode**, **mCoding** | Concise, implementation-level walkthroughs |

---

[← Day 30 — Topological Sort](day-30-topological-sort.md) · [Day 32 — Union-Find →](day-32-union-find.md)
