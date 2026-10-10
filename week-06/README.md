# Week 6 — Production AI Services & Dynamic Programming

[← Curriculum home](../README.md)

**Theme:** This is the last stretch before you own a production AI service. The Python days cover the three things every LLM gateway needs on day one: **observability** (knowing what happened to a specific request), **resilience** (surviving dependencies that fail, slow down or rate-limit you) and **streaming** (getting tokens to users as they're generated, without leaking upstream connections when users leave). The DSA days cover dynamic programming and backtracking, the two patterns behind most "count / optimize / enumerate the ways" problems.

The Python days build on earlier weeks. `contextvars` follows Day 5's tasks, since each task runs in its own context copy. Resilience timeouts rely on Week 4's cancellation semantics. Streaming combines Day 3's generator cleanup, Week 4's backpressure and Week 5's ASGI response model.

| Day | Track | Topic | Problem |
|---|---|---|---|
| [36](day-36-observability-contextvars-otel.md) | Python | Observability: contextvars, structured logs, OpenTelemetry | — |
| [37](day-37-dp-1d.md) | DSA | 1-D Dynamic Programming | LC 322 · Coin Change |
| [38](day-38-resilience-patterns.md) | Python | Resilience: retries, circuit breakers, rate limiting | — |
| [39](day-39-dp-2d.md) | DSA | 2-D Dynamic Programming on two sequences | LC 1143 · Longest Common Subsequence |
| [40](day-40-streaming-llm-responses.md) | Python | Streaming LLM responses (SSE + async generators) | — |
| [41](day-41-backtracking-tries.md) | DSA | Backtracking & Tries | LC 39 · Combination Sum |
| [42](day-42-final-consolidation.md) | Audit | Final capstone consolidation | LC 300 · Longest Increasing Subsequence |

## Exit criteria for Week 6 (and the whole plan)

By Day 42 you should be able to do the following **without notes**:

- [ ] Explain why `threading.local` gives wrong request IDs under asyncio, and how `contextvars` fixes it, including the thread-pool exception.
- [ ] Produce JSON logs where every line carries `request_id` and `trace_id`, and explain how a W3C `traceparent` header carries a trace across services.
- [ ] Implement retry with capped exponential backoff + full jitter, and explain why retries must be limited by idempotency, a deadline and a budget.
- [ ] Draw the circuit-breaker state machine and describe the half-open probe rule.
- [ ] Stream tokens over SSE with heartbeats and bounded buffering, and prove the upstream connection closes when the client disconnects.
- [ ] For any DP problem, state the *state*, *transition*, *base case* and *fill order* before writing code.
- [ ] Write the backtracking template from memory and point to where pruning goes.

---

[← Week 5](../week-05/README.md) · [Curriculum home](../README.md)
