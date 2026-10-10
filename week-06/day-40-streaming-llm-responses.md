# Day 40 — Streaming LLM Responses: SSE, Async Generators, and Disconnect Cleanup

[← Week 6](README.md) · **Track:** Python Systems · **Time box:** 60 min (concept 20 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** Users judge LLM products by **time to first token (TTFT)**, not total latency. Streaming makes a 20-second completion feel instant. It also turns a simple request/response into a long-lived connection with failure modes of its own: proxies that buffer the entire stream, users who close the tab while you keep paying for tokens, and upstream connections that never go back to the pool.

**Prerequisites:** Day 3 (generator cleanup, `aclosing`), Day 5 (tasks), Week 4 (cancellation, backpressure), Week 5 (ASGI).

---

## 1. Core Concept

### 1.1 The data path

```text
Model provider ──HTTP chunked / SSE──► your gateway ──SSE──► reverse proxy / LB ──► browser EventSource
  (upstream async iterator)          (transform, meter,         (must NOT buffer)
                                       moderate, re-encode)
```

Your gateway sits in the middle of the stream. It reads an upstream async iterator of chunks, transforms them, and writes SSE frames downstream. **Every stage has to stream.** A single stage that buffers the whole response (a proxy, gzip middleware, a logging middleware that reads the body) brings TTFT back up to the full response time.

### 1.2 Server-Sent Events wire format

SSE is plain text over a long-lived HTTP response with `Content-Type: text/event-stream`:

```text
event: start                 ← optional event type (default "message")
data: {"model":"m-1"}        ← payload; multi-line payloads = multiple data: lines
                             ← BLANK LINE terminates the event (\n\n)
id: 1                        ← optional; browser resends it as Last-Event-ID on reconnect
data: {"delta":"Hel"}

: keep-alive                 ← line starting with ':' is a comment: ignored by clients,
                               but keeps idle-timeout proxies/LBs from cutting the connection
data: [DONE]
```

Rules that bite: a payload containing a newline must be split into several `data:` lines (or JSON-encoded, which escapes newlines), an event isn't delivered until its blank line arrives, and `retry: <ms>` tells the browser how long to wait before reconnecting. Compared with WebSockets, SSE is one-directional, works over plain HTTP/1.1 and HTTP/2 with no protocol upgrade, and passes through most corporate proxies, which is why most LLM APIs use it.

### 1.3 Async generators as the streaming primitive

An `async def` containing `yield` is an async generator. It's the natural shape for "produce chunks over time while awaiting I/O". In ASGI, a framework's `StreamingResponse(agen)` iterates the generator and sends each chunk as an `http.response.body` message with `more_body=True`.

**Cleanup semantics (Day 3, async version):** when the client disconnects, the server stops iterating, typically by cancelling the response task or calling `agen.aclose()`. That raises `GeneratorExit` (or `CancelledError`) **at the current `yield` or `await`**, so your `finally` blocks run and you can `await` cleanup there. If nothing calls `aclose()`, an abandoned async generator is finalized *later* by the event loop's asyncgen hooks, or not at all. Wrap every upstream stream in `async with aclosing(...)`.

### 1.4 Decouple upstream and downstream with a bounded queue

If you iterate the upstream directly inside your response generator, you can't send heartbeats while waiting on a slow first token. Wrapping `anext(upstream)` in `wait_for` doesn't work either: a timeout **cancels the generator's in-flight step and kills the stream**. The robust structure is:

```text
pump task:   async for tok in upstream: await queue.put(tok)     ← queue is BOUNDED
response:    tok = await wait_for(queue.get(), heartbeat)        ← timeout → emit ": keep-alive"
finally:     pump.cancel(); await pump                           ← cancellation reaches upstream → aclose
```

- **Backpressure:** with `Queue(maxsize=N)`, a slow client fills the queue, so `put()` blocks, the pump stops reading upstream, and TCP flow control slows the provider. Memory per stream stays at O(N), and a stalled client can't make your process buffer an entire completion. Week 4's backpressure lesson applies end to end.
- **TTFT** = time from request start to the first content frame. Measure it **at the gateway** and export it as a histogram, per model. Also record tokens/s and the *reason* each stream ended (complete, client_disconnect, upstream_error, timeout).

---

## 2. Production Gotcha: streams that don't stream, and streams that never stop

### 2a. The proxy buffers everything

Your FastAPI endpoint streams perfectly in local testing. Behind nginx, users see nothing for 15 s and then the entire answer at once. **nginx buffers proxied responses by default** (`proxy_buffering on`). Other buffering stages include gzip/Brotli compression middleware, some API gateways and CDNs, and body-logging middleware.

**Fix:** send `X-Accel-Buffering: no` (nginx honours it per response) or set `proxy_buffering off;` on the location; set `Cache-Control: no-cache`; exclude `text/event-stream` from compression; raise idle timeouts (`proxy_read_timeout`, ALB idle timeout, default 60 s) **and** send heartbeats more often than the shortest idle timeout on the path.

### 2b. The client leaves; the upstream keeps generating

The user closes the tab after 2 tokens. If your code does `async for chunk in provider_stream:` without cleanup wired to disconnects, then depending on the framework and code shape, the upstream request **keeps running to completion**. You pay for all 4,000 output tokens, keep a pooled connection busy and hold a concurrency slot. Multiply that by every user who clicks "Stop generating".

**Fix:** make sure disconnects cancel your response generator (modern Starlette does this; with raw ASGI, watch for `http.disconnect` from `receive()`). Wrap upstream iteration in `aclosing`. Cancel helper tasks in `finally` and await them. Verify with a test that asserts **the upstream was closed early**, exactly like the code below does.

---

## 3. Runnable Code Example

Save as `day40_streaming.py`. It's stdlib only and simulates the full path: a fake model stream, an SSE encoder with heartbeats and a bounded queue, and three clients (normal, disconnecting early, slow).

```python
"""
Day 40 — Streaming LLM tokens over SSE: async generators, bounded-queue backpressure,
heartbeats, client-disconnect cleanup, and time-to-first-token.

Run:  python day40_streaming.py        (Python 3.12+)
"""
from __future__ import annotations

import asyncio
import json
from collections.abc import AsyncIterator
from contextlib import aclosing
from dataclasses import dataclass

TOKENS = ["Stream", "ing", " keeps", " TTFT", " low", ";", " buffering", " proxies", " kill", " it", "."]
HEARTBEAT = b": keep-alive\n\n"  # SSE comment: ignored by clients, resets proxy idle timers


@dataclass(slots=True)
class StreamMetrics:
    ttft_ms: float | None = None
    sent: int = 0
    generated: int = 0
    heartbeats: int = 0
    producer_blocked: int = 0
    end_reason: str = "unknown"


async def fake_llm_stream(m: StreamMetrics, *, prefill_s: float, per_token_s: float) -> AsyncIterator[str]:
    """Stands in for iterating a provider's streaming response (holds an HTTP connection)."""
    try:
        await asyncio.sleep(prefill_s)            # prompt processing before the first token
        for tok in TOKENS:
            m.generated += 1
            yield tok
            await asyncio.sleep(per_token_s)      # decode step
    finally:
        await asyncio.sleep(0)                    # e.g. `await response.aclose()` -> conn back to pool
        print(f"    [upstream] closed after generating {m.generated}/{len(TOKENS)} tokens")


def sse_event(data: str, *, event: str | None = None, event_id: str | None = None) -> bytes:
    lines: list[str] = []
    if event:
        lines.append(f"event: {event}")
    if event_id:
        lines.append(f"id: {event_id}")
    lines.extend(f"data: {part}" for part in data.split("\n"))  # newline-safe payloads
    return ("\n".join(lines) + "\n\n").encode()


async def sse_stream(
    m: StreamMetrics,
    *,
    prefill_s: float = 0.0,
    per_token_s: float = 0.01,
    heartbeat_s: float = 0.1,
    queue_size: int = 4,
) -> AsyncIterator[bytes]:
    """What a StreamingResponse would iterate. Headers to send alongside it:
    Content-Type: text/event-stream, Cache-Control: no-cache, X-Accel-Buffering: no"""
    loop = asyncio.get_running_loop()
    started = loop.time()
    queue: asyncio.Queue[str | None] = asyncio.Queue(maxsize=queue_size)

    async def pump() -> None:
        async with aclosing(fake_llm_stream(m, prefill_s=prefill_s, per_token_s=per_token_s)) as upstream:
            async for tok in upstream:
                if queue.full():
                    m.producer_blocked += 1
                await queue.put(tok)              # bounded: slow client -> upstream reading pauses
        await queue.put(None)                     # end-of-stream sentinel

    producer = asyncio.create_task(pump(), name="upstream-pump")
    try:
        yield sse_event(json.dumps({"model": "fake-1"}), event="start")
        while True:
            try:
                tok = await asyncio.wait_for(queue.get(), timeout=heartbeat_s)
            except TimeoutError:
                if producer.done():
                    producer.result()             # surface an upstream crash; don't heartbeat forever
                m.heartbeats += 1
                yield HEARTBEAT
                continue
            if tok is None:
                break
            if m.ttft_ms is None:
                m.ttft_ms = (loop.time() - started) * 1000
            m.sent += 1
            yield sse_event(json.dumps({"delta": tok}), event_id=str(m.sent))
        yield sse_event("[DONE]", event="end")
        m.end_reason = "complete"
    except GeneratorExit:
        m.end_reason = "client_disconnect"
        raise
    finally:
        # Runs on completion, on error, and on disconnect (aclose -> GeneratorExit at our yield).
        producer.cancel()
        await asyncio.gather(producer, return_exceptions=True)  # wait until upstream is really closed


# ---------------------------------------------------------------------------
# Simulated clients (stand-ins for a browser behind an ASGI server)
# ---------------------------------------------------------------------------
async def client(name: str, m: StreamMetrics, stream: AsyncIterator[bytes], *,
                 show_frames: int = 0, disconnect_after: int | None = None,
                 per_frame_s: float = 0.0) -> None:
    print(f"\n== {name}")
    deltas = i = 0
    async with aclosing(stream) as frames:        # what the server does on disconnect: aclose()
        async for frame in frames:
            i += 1
            if i <= show_frames:
                print("  frame:", frame.decode().replace("\n", "\\n"))
            if frame.startswith(b"id:"):
                deltas += 1
                if disconnect_after is not None and deltas == disconnect_after:
                    print(f"  client: closing tab after {deltas} tokens")
                    break
            if per_frame_s:
                await asyncio.sleep(per_frame_s)  # slow network / slow renderer
    ttft = f"{m.ttft_ms:.0f} ms" if m.ttft_ms is not None else "n/a"
    print(f"  metrics: end={m.end_reason} ttft={ttft} sent={m.sent} generated={m.generated} "
          f"heartbeats={m.heartbeats} producer_blocked={m.producer_blocked}")


async def main() -> None:
    m1 = StreamMetrics()
    await client("normal client; slow prefill (250 ms) > heartbeat (100 ms)", m1,
                 sse_stream(m1, prefill_s=0.25), show_frames=5)

    m2 = StreamMetrics()
    await client("client disconnects after 3 tokens", m2,
                 sse_stream(m2, per_token_s=0.02), disconnect_after=3)

    m3 = StreamMetrics()
    await client("slow client (30 ms/frame) vs instant upstream, queue=4", m3,
                 sse_stream(m3, per_token_s=0.0), per_frame_s=0.03)


if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output** (TTFT varies by a few ms between runs and platforms):

```text
== normal client; slow prefill (250 ms) > heartbeat (100 ms)
  frame: event: start\ndata: {"model": "fake-1"}\n\n
  frame: : keep-alive\n\n
  frame: : keep-alive\n\n
  frame: id: 1\ndata: {"delta": "Stream"}\n\n
  frame: id: 2\ndata: {"delta": "ing"}\n\n
    [upstream] closed after generating 11/11 tokens
  metrics: end=complete ttft=265 ms sent=11 generated=11 heartbeats=2 producer_blocked=0

== client disconnects after 3 tokens
  client: closing tab after 3 tokens
    [upstream] closed after generating 3/11 tokens
  metrics: end=client_disconnect ttft=0 ms sent=3 generated=3 heartbeats=0 producer_blocked=0

== slow client (30 ms/frame) vs instant upstream, queue=4
    [upstream] closed after generating 11/11 tokens
  metrics: end=complete ttft=31 ms sent=11 generated=11 heartbeats=0 producer_blocked=7
```

How to read it:

- **Normal client:** two `: keep-alive` frames are sent during the 250 ms prefill, TTFT ≈ 250 ms, and the upstream closes after generating all 11 tokens.
- **Disconnecting client:** `aclose()` raises `GeneratorExit` inside `sse_stream`, and `finally` cancels the pump. The upstream reports that it closed after generating **3 of 11** tokens, so you stopped paying for the rest.
- **Slow client:** the upstream could produce everything instantly, but `producer_blocked=7` shows the pump waiting on the full 4-slot queue. That's backpressure working. The upstream is paced by the client rather than buffered without limit in memory.

**Exercises (10 min):**

1. Replace `aclosing(stream)` in `client` with plain `async for` + `break`, and drop the reference. Watch when (or whether) `[upstream] closed` prints now.
2. Make `fake_llm_stream` raise `ConnectionError` after 4 tokens. Emit an `event: error` SSE frame to the client and set `end_reason="upstream_error"`.
3. Add `retry: 3000` and `id:` resume support: accept a `last_event_id` and skip tokens the client already has. (Real providers can't resume a generation, so think about what you'd need to cache.)

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `server sent events explained` | **Hussein Nasser** | Protocol-level view of SSE vs WebSockets vs long polling, including proxy behaviour |
| Deep dive | `FastAPI StreamingResponse server sent events LLM` | **ArjanCodes** / **Pretty Printed** | End-to-end streaming endpoints in a real framework |
| Practical | `python async generators aclose` | **anthonywritescode** / **mCoding** | The finalization semantics behind today's disconnect handling |

**Read:** WHATWG HTML spec, section *Server-sent events* (the wire format is short); your model provider's streaming API docs (event types, error events mid-stream); nginx docs for `proxy_buffering` and `X-Accel-Buffering`.

---

[← Day 39 — 2-D DP](day-39-dp-2d.md) · [Day 41 — Backtracking & Tries →](day-41-backtracking-tries.md)
