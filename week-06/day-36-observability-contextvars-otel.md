# Day 36 — Observability: contextvars, Structured Logs, and OpenTelemetry

[← Week 6](README.md) · **Track:** Python Systems · **Time box:** 55 min (concept 20 · gotcha 10 · code 20 · review 5)

> **Why this matters for platform work:** At 3 a.m. the question is never "is the service up?" It's "what happened to *this* user's request, across the gateway, the vector DB and the model provider?" Answering that needs one thing done right everywhere: **request context that follows execution**, even across `await`s, tasks, threads and network hops.

---

## 1. Core Concept

### 1.1 Why `threading.local` is wrong under asyncio

`threading.local()` stores one value **per OS thread**. An asyncio service runs thousands of concurrent requests on **one** thread (Day 5), so every coroutine shares the same "thread-local" slot:

```text
t=0  request A sets tls.request_id = "A"   → await db()   (A suspended)
t=1  request B sets tls.request_id = "B"   → await db()   (B suspended)
t=2  A resumes and logs tls.request_id     → "B"   🚨 wrong request attributed
```

Nothing crashes. Your logs just quietly attribute errors to the wrong customer.

### 1.2 How `contextvars` works

A `ContextVar` is a *key*. Its values live in a `Context`, an **immutable mapping** (a HAMT, hash array mapped trie, so copies are $O(1)$ and updates are $O(\log n)$ with structural sharing).

- Every thread has a *current* `Context`. `var.get()` reads from it, and `var.set(v)` produces a new mapping and makes it current. `set()` returns a `Token`, and `var.reset(token)` restores the previous value. Use this pattern in middleware.
- **asyncio integration:** `asyncio.create_task()` (and `TaskGroup.create_task`) calls `contextvars.copy_context()` **at task creation**, and every step of that task runs inside `ctx.run(...)`. As a result:
  - a child task *inherits* the parent's values at creation time;
  - values the child sets are **invisible to the parent** and to sibling tasks, because each task has its own copy;
  - plain `await coro()` (no new task) runs in the **same** context, so a `set()` inside it *is* visible to the caller afterwards.
- `loop.call_soon(cb, context=ctx)` lets callbacks carry context too. This is how futures restore it.

| Mechanism | Context propagated? |
|---|---|
| `await coro()` | Shared: same context |
| `asyncio.create_task` / `TaskGroup` | Copied at creation |
| `asyncio.to_thread(fn)` | **Copied** (it uses `copy_context().run`) |
| `loop.run_in_executor(None, fn)` | **Not copied**: the worker thread sees defaults |
| `ThreadPoolExecutor.submit(fn)` | **Not copied**. Fix: `pool.submit(copy_context().run, fn)` |
| `multiprocessing` / Celery / queues | Never. Serialize the IDs into the message explicitly |

### 1.3 Structured logging: context goes in via a `Filter`, output comes out of a `Formatter`

```text
logger.info("msg", extra={...})
   → LogRecord created (in the CALLER's context)
   → logger filters → propagate up to ancestors → handler filters  ← stamp request_id here
   → handler.formatter.format(record)                              ← emit JSON here
```

- **Attach the context filter to the handler**, not to a logger. Filters on a logger run only for records logged *directly on that logger*. They're skipped for records propagated up from child loggers like `gateway.db` or `httpx`.
- The filter runs synchronously in the logging call, so `ContextVar.get()` sees the right request.
- Emit **one JSON object per line** with stable keys (`level`, `logger`, `msg`, `request_id`, `trace_id`, plus domain fields from `extra=`). Your log pipeline (Loki, CloudWatch, Datadog, ELK) indexes keys, not prose.

### 1.4 OpenTelemetry in one page

| Concept | Meaning |
|---|---|
| **Trace** | One end-to-end request across all services. Identified by a 16-byte `trace_id` |
| **Span** | One timed operation inside a trace (HTTP handler, DB query, LLM call). It has an 8-byte `span_id`, a parent span, attributes and events |
| **Context propagation** | Carries the *current span* in-process (OTel's Python SDK stores it **in a `ContextVar`**, so today's mechanics apply directly) and across processes via headers |
| **W3C `traceparent`** | `version-traceid-parentspanid-flags`, e.g. `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01` (`01` = sampled) |
| **Exporter / Collector** | The SDK batches spans and exports them over OTLP to a Collector, which samples and routes them to Jaeger, Tempo, Honeycomb or Datadog |

Illustration (not runnable without `opentelemetry-sdk`):

```python
from opentelemetry import trace

tracer = trace.get_tracer("gateway")

async def complete(prompt: str) -> str:
    with tracer.start_as_current_span("llm.complete") as span:   # becomes current via a ContextVar
        span.set_attribute("llm.model", "model-x")
        span.set_attribute("llm.prompt_tokens", count_tokens(prompt))
        resp = await http.post(URL, json=..., headers=inject_traceparent())  # child service continues the trace
        span.set_attribute("llm.completion_tokens", resp.usage.output_tokens)
        return resp.text
```

**Correlate logs and traces:** put `trace_id` into every log line (OTel's logging instrumentation does this). Then a slow span in Jaeger leads straight to its logs, and the other way round.

---

## 2. Production Gotcha: context that silently disappears at the thread boundary

```python
async def handler(req):
    request_id.set(req.headers["x-request-id"])
    loop = asyncio.get_running_loop()
    await loop.run_in_executor(None, sync_pdf_parse, req.body)  # 🚨 logs inside show request_id="-"
```

`run_in_executor` hands the function to a worker thread **without copying the context**. Every log line, metric tag and span emitted inside `sync_pdf_parse` loses its request ID. The same applies to callbacks registered on third-party thread pools, gRPC sync stubs and SDKs with background threads. You only notice during an incident, when the one log line you need has `request_id: "-"`.

**Fixes:**

1. Prefer `await asyncio.to_thread(fn, ...)`, which copies the context.
2. If you need a specific executor: `loop.run_in_executor(pool, contextvars.copy_context().run, fn, *args)`.
3. For process pools and task queues, pass `request_id` and `traceparent` **explicitly** in the payload and re-`set()` them on the worker side.

**Related traps:**

- **f-strings in log calls** (`log.debug(f"payload={big_dict}")`) build the string *even when DEBUG is disabled*, so you pay a `repr` of a multi-MB dict on every request. Use `log.debug("payload=%s", big_dict)`, which formats lazily, or guard with `if log.isEnabledFor(logging.DEBUG):`.
- **Middleware that `set()`s without `reset()`**: in thread-based servers (WSGI) the next request on that thread inherits the stale value. Always `token = var.set(v)` … `finally: var.reset(token)`.
- **Logging secrets:** a JSON formatter that dumps `extra=` blindly will happily log `Authorization` headers and prompts that contain PII. Use an allowlist of keys or a redaction filter.

---

## 3. Runnable Code Example

Save as `day36_observability.py`. It runs three concurrent requests with correct per-request context, compares them with `threading.local`, shows task isolation, and demonstrates the thread-pool gotcha and its fixes.

```python
"""
Day 36 — Request-scoped context with contextvars + structured JSON logs.

Run:  python day36_observability.py        (Python 3.12+)
"""
from __future__ import annotations

import asyncio
import contextvars
import json
import logging
import sys
import threading
from typing import Any

request_id: contextvars.ContextVar[str] = contextvars.ContextVar("request_id", default="-")
trace_id: contextvars.ContextVar[str] = contextvars.ContextVar("trace_id", default="-")

# Attributes every LogRecord has; anything else on the record came from `extra=` or our filter.
_STD_ATTRS = frozenset(vars(logging.LogRecord("x", 0, "x", 0, "x", None, None))) | {"message"}


class ContextFilter(logging.Filter):
    """Stamps each record with the caller's request context (runs synchronously at log time)."""

    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id.get()
        record.trace_id = trace_id.get()
        return True


class JsonFormatter(logging.Formatter):
    """One JSON object per line. Production: add "ts" (record.created), service, version, host."""

    def format(self, record: logging.LogRecord) -> str:
        payload: dict[str, Any] = {
            "level": record.levelname,
            "logger": record.name,
            "msg": record.getMessage(),  # %-style args formatted here, lazily
        }
        payload.update({k: v for k, v in vars(record).items() if k not in _STD_ATTRS})
        if record.exc_info:
            payload["exc"] = self.formatException(record.exc_info)
        return json.dumps(payload, default=str)


def configure_logging() -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())
    handler.addFilter(ContextFilter())  # on the HANDLER, so child loggers' records get stamped too
    root = logging.getLogger()
    root.handlers[:] = [handler]
    root.setLevel(logging.INFO)


log = logging.getLogger("gateway")
_tls = threading.local()  # the WRONG tool, kept for comparison


def parse_traceparent(header: str) -> str:
    """W3C traceparent: version-traceid-spanid-flags. Returns the trace id."""
    version, tid, _span, _flags = header.split("-")
    if version != "00" or len(tid) != 32:
        raise ValueError(f"bad traceparent: {header!r}")
    return tid


# ---------------------------------------------------------------------------
# 1. Concurrent requests: contextvars vs threading.local
# ---------------------------------------------------------------------------
async def vector_db_query(delay: float) -> None:
    await asyncio.sleep(delay)
    logging.getLogger("gateway.vectordb").info("query ok", extra={"latency_ms": int(delay * 1000)})


async def handle_request(rid: str, traceparent: str, delay: float) -> None:
    request_id.set(rid)               # safe: this task runs in its own context copy
    trace_id.set(parse_traceparent(traceparent))
    _tls.request_id = rid             # BUG: one slot shared by every coroutine on this thread
    log.info("request start")
    await vector_db_query(delay)
    log.info("request done", extra={"tls_request_id": _tls.request_id})


async def demo_concurrency() -> None:
    print("-- 1. three concurrent requests (note tls_request_id on 'done' lines)")
    requests = [
        ("req-A", "00-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa-1111111111111111-01", 0.03),
        ("req-B", "00-bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb-2222222222222222-01", 0.01),
        ("req-C", "00-cccccccccccccccccccccccccccccccc-3333333333333333-01", 0.02),
    ]
    async with asyncio.TaskGroup() as tg:
        for rid, tp, delay in requests:
            tg.create_task(handle_request(rid, tp, delay))


# ---------------------------------------------------------------------------
# 2. Isolation: child TASK gets a copy; a plain AWAIT shares the context
# ---------------------------------------------------------------------------
async def overwrite() -> None:
    request_id.set("overwritten-by-callee")


async def demo_isolation() -> None:
    print("\n-- 2. task copy vs shared await")
    request_id.set("req-parent")
    await asyncio.create_task(overwrite())
    log.info("after awaiting a child TASK")       # still req-parent
    await overwrite()
    log.info("after a plain AWAIT")               # overwritten-by-callee


# ---------------------------------------------------------------------------
# 3. The thread-boundary gotcha
# ---------------------------------------------------------------------------
def sync_work(via: str) -> None:
    log.info("inside worker thread", extra={"via": via})


async def demo_threads() -> None:
    print("\n-- 3. crossing into threads")
    request_id.set("req-thread")
    loop = asyncio.get_running_loop()
    await loop.run_in_executor(None, sync_work, "run_in_executor")              # context LOST
    await asyncio.to_thread(sync_work, "to_thread")                             # copied
    ctx = contextvars.copy_context()
    await loop.run_in_executor(None, ctx.run, sync_work, "executor+ctx.run")    # copied


async def main() -> None:
    configure_logging()
    # Each demo runs in its own task so its set() calls don't leak into the next demo.
    for demo in (demo_concurrency, demo_isolation, demo_threads):
        await asyncio.create_task(demo())


if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output:**

```text
-- 1. three concurrent requests (note tls_request_id on 'done' lines)
{"level": "INFO", "logger": "gateway", "msg": "request start", "request_id": "req-A", "trace_id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"}
{"level": "INFO", "logger": "gateway", "msg": "request start", "request_id": "req-B", "trace_id": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"}
{"level": "INFO", "logger": "gateway", "msg": "request start", "request_id": "req-C", "trace_id": "cccccccccccccccccccccccccccccccc"}
{"level": "INFO", "logger": "gateway.vectordb", "msg": "query ok", "latency_ms": 10, "request_id": "req-B", "trace_id": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"}
{"level": "INFO", "logger": "gateway", "msg": "request done", "tls_request_id": "req-C", "request_id": "req-B", "trace_id": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"}
{"level": "INFO", "logger": "gateway.vectordb", "msg": "query ok", "latency_ms": 20, "request_id": "req-C", "trace_id": "cccccccccccccccccccccccccccccccc"}
{"level": "INFO", "logger": "gateway", "msg": "request done", "tls_request_id": "req-C", "request_id": "req-C", "trace_id": "cccccccccccccccccccccccccccccccc"}
{"level": "INFO", "logger": "gateway.vectordb", "msg": "query ok", "latency_ms": 30, "request_id": "req-A", "trace_id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"}
{"level": "INFO", "logger": "gateway", "msg": "request done", "tls_request_id": "req-C", "request_id": "req-A", "trace_id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"}

-- 2. task copy vs shared await
{"level": "INFO", "logger": "gateway", "msg": "after awaiting a child TASK", "request_id": "req-parent", "trace_id": "-"}
{"level": "INFO", "logger": "gateway", "msg": "after a plain AWAIT", "request_id": "overwritten-by-callee", "trace_id": "-"}

-- 3. crossing into threads
{"level": "INFO", "logger": "gateway", "msg": "inside worker thread", "via": "run_in_executor", "request_id": "-", "trace_id": "-"}
{"level": "INFO", "logger": "gateway", "msg": "inside worker thread", "via": "to_thread", "request_id": "req-thread", "trace_id": "-"}
{"level": "INFO", "logger": "gateway", "msg": "inside worker thread", "via": "executor+ctx.run", "request_id": "req-thread", "trace_id": "-"}
```

Read it line by line. `request_id` is always right. `tls_request_id` says `req-C` for **all three** requests. The `gateway.vectordb` child logger's records are stamped because the filter sits on the handler. The plain `run_in_executor` line has lost its request ID.

**Exercises (10 min):**

1. Move `addFilter(ContextFilter())` from the handler to `logging.getLogger("gateway")`. The `gateway.vectordb` lines lose `request_id`: the filter no longer sees them.
2. Write an ASGI-style middleware function: `token = request_id.set(header or uuid4().hex)`, call the app, then `request_id.reset(token)` in `finally`. Also echo the ID in an `x-request-id` response header.
3. Add an outgoing-header helper `make_traceparent() -> str` that keeps the current `trace_id` and generates a new 8-byte span id with `secrets.token_hex(8)`.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `python contextvars asyncio explained` | **anthonywritescode** / **mCoding** | How context copying works per task, with live demos |
| Deep dive | `OpenTelemetry Python tracing tutorial` | **CNCF** / **OpenTelemetry** channel | Spans, propagation and the Collector pipeline from the maintainers |
| Practical | `structured logging python json` | **ArjanCodes** | Clean logging architecture and why string logs don't scale |

**Read:** PEP 567 (contextvars) and the W3C Trace Context specification (`traceparent`/`tracestate`). Both are short and precise.

---

[← Day 35 — Consolidation](../week-05/day-35-consolidation.md) · [Day 37 — 1-D DP →](day-37-dp-1d.md)
