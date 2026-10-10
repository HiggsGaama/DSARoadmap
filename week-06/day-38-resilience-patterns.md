# Day 38 — Resilience Patterns: Retries, Deadlines, Circuit Breakers, Rate Limits

[← Week 6](README.md) · **Track:** Python Systems · **Time box:** 60 min (concept 20 · gotcha 10 · code 25 · review 5)

> **Why this matters for platform work:** An AI gateway is a thin layer over unreliable dependencies: model providers return 429 and 529, vector DBs hit GC pauses, embedding services time out. Naive retries make outages **worse**. The patterns below decide whether a 30-second provider blip becomes a 30-second blip for your users or a 30-minute self-inflicted retry storm.

**Prerequisites:** Day 5 (event loop, `asyncio.timeout`), Week 4 (cancellation semantics).

---

## 1. Core Concept

### 1.1 What to retry, and what never to retry

| Retry? | Errors | Why |
|---|---|---|
| ✅ Yes | Connection reset or refused, timeouts, `502/503/504`, `429` (honour `Retry-After`), provider "overloaded" errors | Transient: the same request may succeed later |
| ❌ No | `400`, `401`, `403`, `404`, `422`, schema or validation errors, content-policy refusals | Deterministic: retrying sends the same bad request again and wastes quota |
| ⚠️ Only if idempotent | Anything on `POST` that **creates or charges** (payments, sending an email, starting a fine-tuning job) | The first attempt may have succeeded and only the *response* got lost. Make it safe with an **idempotency key** the server deduplicates on |

### 1.2 Backoff with full jitter

Retrying immediately or at fixed intervals synchronizes every client: they all failed at the same moment, so they all retry at the same moment ("thundering herd"). Use **capped exponential backoff with full jitter**:

$$\text{sleep} = \text{uniform}(0,\ \min(\text{cap},\ \text{base} \cdot 2^{\text{attempt}}))$$

"Full jitter" randomizes over the *whole* interval, which spreads retries out the most (AWS Architecture Blog, *Exponential Backoff and Jitter*). If the server sends `Retry-After`, it overrides your computed delay.

### 1.3 Deadlines, not per-call timeouts alone

A request has **one deadline** (for example, 10 s for the user-facing call). Each attempt's timeout is `min(attempt_timeout, deadline − now)`, and you **never start a retry you can't finish**. Propagate the remaining budget downstream (gRPC does this natively, and HTTP services usually use a header such as `x-request-deadline`). Without that, a downstream service keeps working on a request the user abandoned long ago.

### 1.4 Retry amplification and retry budgets

With 3 attempts per layer and 3 layers (gateway → retrieval → embedding), one user request can become $3^3 = 27$ requests on the bottom service, at exactly the moment it's struggling. Defences:

- **Retry at one layer only**, ideally the one closest to the user that knows the deadline.
- **Retry budget:** each request deposits a fraction of a token (for example 0.2), and each retry spends a whole token. Retries are then capped at about 20% of traffic regardless of how many requests fail. This idea comes from Finagle and gRPC's retry throttling.

### 1.5 Circuit breaker: stop calling a dependency that's down

```text
            failures >= threshold
   CLOSED ─────────────────────────► OPEN ──── (reset_timeout elapsed) ───► HALF_OPEN
     ▲  (calls pass; count failures)    (fail fast, no calls)                 (allow ONE probe)
     │                                                                        │      │
     └─────────────────────── probe succeeds ─────────────────────────────────┘      │
                                     OPEN ◄──────────── probe fails ─────────────────┘
```

- **CLOSED:** calls flow through and consecutive failures are counted.
- **OPEN:** fail immediately with `CircuitOpenError`. The dependency gets breathing room, and your request handlers don't pile up waiting on timeouts. Serve a fallback (cached answer, smaller model, "degraded" response) if you have one.
- **HALF_OPEN:** allow **exactly one** probe. Success closes the circuit and failure re-opens it. Allowing many concurrent probes recreates the stampede.
- Count only **dependency failures**. A `400` means the dependency is healthy and *you* sent garbage.

### 1.6 Token bucket: limit your own rate before the provider does it for you

The bucket holds up to `capacity` tokens and refills at `rate` tokens per second. Each call takes one token, and if none are available it waits (or gets rejected). `capacity` allows bursts and `rate` sets the long-term throughput. Providers enforce requests-per-minute *and* tokens-per-minute limits, so production gateways usually run **two** buckets, one per limit, and weight the token bucket by the estimated prompt plus completion tokens.

---

## 2. Production Gotcha: the retry loop that amplified an outage

```python
async def call_model(payload: dict) -> dict:
    for attempt in range(5):
        try:
            return await client.post("/v1/messages", json=payload, timeout=30)
        except Exception:                     # 🚨 retries 400s, auth errors, even bugs in our own code
            await asyncio.sleep(1)            # 🚨 fixed delay: every pod retries in lockstep
    raise RuntimeError("model unavailable")   # 🚨 5 × 30 s = 150 s worst case, long after the user left
```

A provider has a 60-second brownout. Every request now makes 5 attempts, so load on the provider and on your connection pool **quintuples**. The fixed 1-second delay synchronizes all pods. Your workers hold sockets for up to 150 s each, so the pool runs dry, and healthy routes (health checks, other models) start failing too. **The outage spreads to everything your service does.**

**Fix checklist:**

1. Retry only on retryable errors (§1.1). Let `CancelledError` propagate; never catch `BaseException`.
2. Use capped exponential backoff with full jitter, and honour `Retry-After`.
3. Set an overall deadline: every attempt is bounded by the remaining budget, and you don't retry without time left.
4. Use a retry budget, and retry at a single layer.
5. Put a circuit breaker in front of each dependency (per provider and model), plus a fallback.
6. For non-idempotent calls, use an idempotency key or no retries at all.

Libraries such as `tenacity` (retries), `aiobreaker` / `pybreaker` (breakers) and `aiolimiter` (rate limits) implement these, but you should understand the state machines well enough to configure them, which is the point of today's code.

---

## 3. Runnable Code Example

Save as `day38_resilience.py`. It's stdlib only and runs a scripted flaky dependency through each pattern.

```python
"""
Day 38 — Resilience primitives: retry with full jitter + budget + deadline,
circuit breaker, and token-bucket rate limiting.

Run:  python day38_resilience.py        (Python 3.12+)
"""
from __future__ import annotations

import asyncio
import random
import time
from collections import deque
from collections.abc import Awaitable, Callable
from dataclasses import dataclass, field
from enum import Enum


class RetryableError(Exception):
    """5xx, 429, connection resets: the same request may succeed later."""


class NonRetryableError(Exception):
    """4xx: the request itself is wrong; retrying cannot help."""


class CircuitOpenError(Exception):
    """Raised without calling the dependency."""


# ---------------------------------------------------------------------------
# Retry: full jitter + per-attempt timeout bounded by a deadline + budget
# ---------------------------------------------------------------------------
@dataclass(frozen=True, slots=True)
class RetryPolicy:
    max_attempts: int = 4
    base_delay: float = 0.02
    max_delay: float = 0.5
    attempt_timeout: float = 0.1

    def backoff(self, attempt: int, rng: random.Random) -> float:
        # Full jitter: uniform(0, min(cap, base * 2^attempt))
        return rng.uniform(0, min(self.max_delay, self.base_delay * 2**attempt))


@dataclass(slots=True)
class RetryBudget:
    """Each request deposits `ratio` tokens; each retry spends 1. Caps retries at ~ratio of traffic."""

    ratio: float = 0.2
    max_tokens: float = 10.0
    tokens: float = 10.0

    def on_request(self) -> None:
        self.tokens = min(self.max_tokens, self.tokens + self.ratio)

    def try_spend(self) -> bool:
        if self.tokens >= 1.0:
            self.tokens -= 1.0
            return True
        return False


async def retry_call[T](
    op: Callable[[], Awaitable[T]],
    *,
    policy: RetryPolicy,
    budget: RetryBudget,
    deadline: float,
    idempotent: bool,
    rng: random.Random,
) -> T:
    loop = asyncio.get_running_loop()
    budget.on_request()
    attempts = policy.max_attempts if idempotent else 1  # never blindly retry non-idempotent calls
    last_exc: Exception | None = None

    for attempt in range(attempts):
        remaining = deadline - loop.time()
        if remaining <= 0:
            break
        try:
            async with asyncio.timeout(min(policy.attempt_timeout, remaining)):
                return await op()
        except NonRetryableError:
            raise                                     # 4xx: surface immediately
        except (RetryableError, TimeoutError) as exc:
            last_exc = exc
        # CircuitOpenError / CancelledError / bugs are NOT caught: they propagate untouched.

        if attempt == attempts - 1:
            break
        if not budget.try_spend():
            print("    retry budget exhausted -> giving up early")
            break
        pause = policy.backoff(attempt, rng)
        if loop.time() + pause >= deadline:
            print("    not enough deadline left for another attempt")
            break
        print(f"    attempt {attempt + 1} failed ({last_exc!r}); backoff {pause * 1000:.0f} ms")
        await asyncio.sleep(pause)

    if last_exc is None:
        raise TimeoutError("deadline exceeded before first attempt")
    raise last_exc


# ---------------------------------------------------------------------------
# Circuit breaker
# ---------------------------------------------------------------------------
class State(Enum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"


@dataclass(slots=True)
class CircuitBreaker:
    name: str
    failure_threshold: int = 3
    reset_timeout: float = 0.2
    clock: Callable[[], float] = time.monotonic
    state: State = State.CLOSED
    failures: int = 0
    opened_at: float = 0.0
    probe_in_flight: bool = False

    def _transition(self, new: State) -> None:
        print(f"    [breaker:{self.name}] {self.state.value} -> {new.value}")
        self.state = new
        if new is State.OPEN:
            self.opened_at = self.clock()

    def _before_call(self) -> None:
        if self.state is State.OPEN:
            if self.clock() - self.opened_at < self.reset_timeout:
                raise CircuitOpenError(f"{self.name}: open, failing fast")
            self._transition(State.HALF_OPEN)
        if self.state is State.HALF_OPEN:
            if self.probe_in_flight:
                raise CircuitOpenError(f"{self.name}: half-open, probe in flight")
            self.probe_in_flight = True

    def _on_success(self) -> None:
        self.failures = 0
        if self.state is not State.CLOSED:
            self._transition(State.CLOSED)

    def _on_failure(self) -> None:
        if self.state is State.HALF_OPEN:
            self._transition(State.OPEN)
            return
        self.failures += 1
        if self.failures >= self.failure_threshold:
            self._transition(State.OPEN)

    async def call[T](self, op: Callable[[], Awaitable[T]]) -> T:
        self._before_call()
        try:
            result = await op()
        except NonRetryableError:
            self._on_success()   # dependency answered; the REQUEST was bad
            raise
        except Exception:
            self._on_failure()
            raise
        finally:
            self.probe_in_flight = False  # also on cancellation, or the breaker wedges half-open
        self._on_success()
        return result


# ---------------------------------------------------------------------------
# Token bucket
# ---------------------------------------------------------------------------
class TokenBucket:
    def __init__(self, rate: float, capacity: float) -> None:
        self.rate, self.capacity = rate, capacity
        self._tokens = capacity
        self._updated = time.monotonic()
        self._lock = asyncio.Lock()  # waiters queue FIFO on the lock

    def _refill(self) -> None:
        now = time.monotonic()
        self._tokens = min(self.capacity, self._tokens + (now - self._updated) * self.rate)
        self._updated = now

    async def acquire(self) -> None:
        async with self._lock:
            while True:
                self._refill()
                if self._tokens >= 1:
                    self._tokens -= 1
                    return
                await asyncio.sleep((1 - self._tokens) / self.rate)


# ---------------------------------------------------------------------------
# Scripted fake dependency
# ---------------------------------------------------------------------------
@dataclass(slots=True)
class FlakyService:
    script: deque[str] = field(default_factory=deque)  # outcomes consumed per call; default "ok"
    calls: int = 0

    async def call(self) -> str:
        self.calls += 1
        await asyncio.sleep(0.005)
        outcome = self.script.popleft() if self.script else "ok"
        match outcome:
            case "ok":
                return f"200 OK (call #{self.calls})"
            case "503":
                raise RetryableError("503")
            case "400":
                raise NonRetryableError("400 bad request")
            case "hang":
                await asyncio.sleep(10)
                return "never"
            case _:
                raise AssertionError(outcome)


async def scenario(title: str, svc: FlakyService, *, idempotent: bool = True,
                   breaker: CircuitBreaker | None = None, budget: RetryBudget | None = None) -> None:
    loop = asyncio.get_running_loop()
    rng = random.Random(5)
    op: Callable[[], Awaitable[str]] = svc.call
    if breaker is not None:
        b = breaker
        op = lambda: b.call(svc.call)  # noqa: E731  breaker INSIDE retry: each attempt is checked
    try:
        result = await retry_call(op, policy=RetryPolicy(), budget=budget or RetryBudget(),
                                  deadline=loop.time() + 1.0, idempotent=idempotent, rng=rng)
        print(f"  {title}: {result}  [service calls={svc.calls}]")
    except Exception as exc:
        print(f"  {title}: FAILED {exc!r}  [service calls={svc.calls}]")


async def main() -> None:
    print("== retries")
    await scenario("transient 503s", FlakyService(deque(["503", "503"])))
    await scenario("hung call", FlakyService(deque(["hang"])))
    await scenario("400 is not retried", FlakyService(deque(["400"])))
    await scenario("non-idempotent POST", FlakyService(deque(["503"])), idempotent=False)
    await scenario("budget nearly empty", FlakyService(deque(["503"] * 5)),
                   budget=RetryBudget(tokens=1.0))

    print("\n== circuit breaker (dependency fully down, then recovers)")
    svc = FlakyService(deque(["503"] * 6))
    breaker = CircuitBreaker("model-api")
    for i in range(3):
        await scenario(f"request {i}", svc, breaker=breaker)
    print("  ...waiting for reset_timeout; dependency recovers")
    svc.script.clear()
    await asyncio.sleep(0.25)
    await scenario("request 3 (probe)", svc, breaker=breaker)

    print("\n== token bucket: rate=20/s, capacity=5, 10 calls")
    bucket = TokenBucket(rate=20, capacity=5)
    start = time.monotonic()
    stamps: list[float] = []
    for _ in range(10):
        await bucket.acquire()
        stamps.append(time.monotonic() - start)
    print("  t(s) =", [round(t, 2) for t in stamps])


if __name__ == "__main__":
    asyncio.run(main())
```

**Expected output** (backoff values are deterministic because the RNG is seeded; token-bucket times vary by a few ms, especially on Windows with its ~15 ms timer resolution):

```text
== retries
    attempt 1 failed (RetryableError('503')); backoff 12 ms
    attempt 2 failed (RetryableError('503')); backoff 30 ms
  transient 503s: 200 OK (call #3)  [service calls=3]
    attempt 1 failed (TimeoutError()); backoff 12 ms
  hung call: 200 OK (call #2)  [service calls=2]
  400 is not retried: FAILED NonRetryableError('400 bad request')  [service calls=1]
  non-idempotent POST: FAILED RetryableError('503')  [service calls=1]
    attempt 1 failed (RetryableError('503')); backoff 12 ms
    retry budget exhausted -> giving up early
  budget nearly empty: FAILED RetryableError('503')  [service calls=2]

== circuit breaker (dependency fully down, then recovers)
    attempt 1 failed (RetryableError('503')); backoff 12 ms
    attempt 2 failed (RetryableError('503')); backoff 30 ms
    [breaker:model-api] CLOSED -> OPEN
    attempt 3 failed (RetryableError('503')); backoff 64 ms
  request 0: FAILED CircuitOpenError('model-api: open, failing fast')  [service calls=3]
  request 1: FAILED CircuitOpenError('model-api: open, failing fast')  [service calls=3]
  request 2: FAILED CircuitOpenError('model-api: open, failing fast')  [service calls=3]
  ...waiting for reset_timeout; dependency recovers
    [breaker:model-api] OPEN -> HALF_OPEN
    [breaker:model-api] HALF_OPEN -> CLOSED
  request 3 (probe): 200 OK (call #4)  [service calls=4]

== token bucket: rate=20/s, capacity=5, 10 calls
  t(s) = [0.0, 0.0, 0.0, 0.0, 0.0, 0.06, 0.11, 0.16, 0.2, 0.25]
```

How to read it:

- **Transient 503s:** two jittered backoffs, then success on call #3.
- **Hung call:** the 100 ms per-attempt timeout turns a 10 s hang into one retry.
- **400** and the **non-idempotent POST:** exactly one service call each.
- **Budget:** one retry was allowed, then the budget ran out, so there was no further load on a failing service.
- **Breaker:** request 0's third consecutive failure trips the breaker. Its fourth attempt is rejected with `CircuitOpenError` without calling the service, and that error isn't retried. Requests 1 and 2 also fail fast: `service calls` stays at 3. After the reset timeout, a single probe closes the circuit again.
- **Token bucket:** a burst of 5 at t≈0, then one call every ~50 ms.

**Exercises (10 min):**

1. Make the breaker count `NonRetryableError` as a failure. Send five `400`s and watch a *healthy* dependency get cut off.
2. Add `Retry-After` support: let `RetryableError` carry `retry_after: float | None`, and use `max(backoff, retry_after)`.
3. Remove `finally: self.probe_in_flight = False`, cancel a probe mid-flight with `asyncio.timeout`, and observe the breaker stuck in HALF_OPEN forever.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `circuit breaker pattern explained microservices` | **ByteByteGo** / **Hussein Nasser** | State machine and failure modes, explained visually |
| Deep dive | `AWS re:Invent timeouts retries backoff jitter` | **AWS Events** | The Amazon Builders' Library material on retries, budgets and jitter, from people who run it at scale |
| Practical | `python tenacity retry tutorial` | **ArjanCodes** / **mCoding** | Production retry decorators: what to configure and what to avoid |

**Read:** AWS Architecture Blog *Exponential Backoff and Jitter*; Amazon Builders' Library *Timeouts, retries, and backoff with jitter*; Google SRE Book, chapter *Handling Overload* (retry budgets, client-side throttling).

---

[← Day 37 — 1-D DP](day-37-dp-1d.md) · [Day 39 — 2-D DP →](day-39-dp-2d.md)
