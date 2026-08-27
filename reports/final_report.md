# Day 10 Reliability Report

**Họ và tên:** Đặng Nguyên Giáp — **MSSV:** 2A202601486

## 1. Architecture summary

The gateway routes every request through a cache-first, breaker-guarded provider chain, falling
back to a static degraded response only if the cache misses and every provider is unavailable.

```
User Request
    |
    v
[Gateway.complete(prompt)]
    |
    v
[Cache check: ResponseCache / SharedRedisCache]
    |-- similarity >= threshold & not false-hit --> HIT --> return cached text (route="cache_hit:<score>")
    |
    v MISS (or cache disabled)
[Circuit Breaker: primary] --(CLOSED/HALF_OPEN, allow)--> FakeLLMProvider("primary").complete()
    |  success --> cache.set() --> return (route="primary")
    |  failure / CircuitOpenError (OPEN, timeout not elapsed) --> record_failure(), try next provider
    v
[Circuit Breaker: backup] --(CLOSED/HALF_OPEN, allow)--> FakeLLMProvider("backup").complete()
    |  success --> cache.set() --> return (route="fallback")
    |  failure / CircuitOpenError --> record_failure(), no more providers
    v
[Static fallback]
    return "The service is temporarily degraded. Please try again soon." (route="static_fallback")
```

Circuit breaker state machine (per provider, independent instances in `gateway.breakers`):

```
        failure_count >= failure_threshold
   CLOSED ---------------------------------> OPEN
     ^                                          |
     | success_count >= success_threshold       | reset_timeout_seconds elapsed
     |                                          v
   HALF_OPEN <-------------------------------- (probe allowed)
     |
     +-- probe fails --> OPEN (reason="probe_failure")
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | 3 consecutive failures is enough signal to stop hammering a degrading provider without over-reacting to a single blip; matches the "no retry storm" requirement. |
| reset_timeout_seconds | 2 | Short enough that a recovered provider is probed quickly (recovery_time_ms observed ≈ 2.2s, close to this floor), long enough to avoid immediately re-opening on a still-failing provider. |
| success_threshold | 1 | A single successful probe in HALF_OPEN is enough to trust `FakeLLMProvider`, since failures are memoryless (random per call) rather than a real warm-up cost. |
| cache TTL (ttl_seconds) | 300 | 5 minutes balances staleness risk against hit rate for a FAQ-style workload where answers don't change every request. |
| similarity_threshold | 0.92 | Tested 0.85 first — got false hits on date-varying queries ("refund policy 2024" vs "2026") scoring ~0.87 under n-gram cosine because only the 4-digit year token differs; 0.92 keeps genuine near-duplicates while still requiring the false-hit year/number guard as a second line of defense. |
| load_test.requests | 100 (per scenario, 400 total across 4 scenarios) | Large enough sample for stable P95/P99 percentile estimates and for the circuit breaker to cycle through several open/half-open/closed transitions within one run. |

## 3. SLO definitions

Values below are the **combined** run across all 4 chaos scenarios (`reports/metrics.json`) — this
deliberately includes scenarios engineered to fail (`both_degraded`), so combined SLIs look worse
than any single healthy scenario. See §7 for the per-scenario breakdown, which is the more honest
picture of production-like health.

| SLI | SLO target | Actual value (combined) | Met? |
|---|---|---:|---|
| Availability | >= 99% | 84.5% | ❌ No (dragged down by `both_degraded`) |
| Latency P95 | < 2500 ms | 315.82 ms | ✅ Yes |
| Fallback success rate | >= 95% | 52.67% | ❌ No (dragged down by `both_degraded`, which has no healthy fallback) |
| Cache hit rate | >= 10% | 55.25% | ✅ Yes |
| Recovery time | < 5000 ms | 2234.68 ms | ✅ Yes |

## 4. Metrics

From `reports/metrics.json` (combined run, cache enabled, 400 requests across all scenarios):

| Metric | Value |
|---|---:|
| total_requests | 400 |
| availability | 0.845 |
| error_rate | 0.155 |
| latency_p50_ms | 267.33 |
| latency_p95_ms | 315.82 |
| latency_p99_ms | 320.37 |
| fallback_success_rate | 0.5267 |
| cache_hit_rate | 0.5525 |
| circuit_open_count | 12 |
| recovery_time_ms | 2234.68 |
| estimated_cost | 0.048626 |
| estimated_cost_saved | 0.221 |

## 5. Cache comparison

Same config and scenarios, `cache.enabled: true` vs `cache.enabled: false`
(`configs/default.yaml` vs `configs/no_cache.yaml`, outputs `reports/metrics.json` vs
`reports/metrics_no_cache.json`):

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| availability | 0.7375 | 0.845 | +0.1075 (cache avoids calls that would have hit failing providers) |
| latency_p50_ms | 273.25 | 267.33 | -5.92 ms (cache hits return in 0ms, pulling the median down) |
| latency_p95_ms | 316.15 | 315.82 | ~flat (P95 is dominated by provider latency on cache misses) |
| circuit_open_count | 26 | 12 | -14 (fewer live provider calls means fewer failures accumulate against the breaker) |
| estimated_cost | 0.132756 | 0.048626 | -0.084130 (≈63% cheaper) |
| cache_hit_rate | 0 | 0.5525 | +0.5525 |
| estimated_cost_saved | 0 | 0.221 | +0.221 |

Takeaway: the cache is not just a latency optimization here — because it intercepts requests
before they reach the provider chain, it also reduces the number of failures the circuit breakers
see, which lowers `circuit_open_count` and raises overall availability.

## 6. Redis shared cache

- Why in-memory cache is insufficient for multi-instance deployments: `ResponseCache` keeps
  `self._entries` as a Python list local to one process. If the gateway is scaled to N replicas
  behind a load balancer, each replica has its own empty cache — the effective hit rate stays low
  and identical queries are answered N times by the (expensive, failure-prone) provider chain
  instead of once.
- How `SharedRedisCache` solves this: every entry is written to Redis as a hash
  (`{prefix}{md5(query)}` → `{query, response}`) with a TTL via `EXPIRE`. Any gateway instance
  pointed at the same `redis_url` can `HGET`/`SCAN` the same keys, so a cache warmed by one
  instance is immediately visible to all others — no extra coordination needed.

### Evidence of shared state

Two independent `SharedRedisCache` Python objects (simulating two gateway instances) against the
same Redis instance:

```
>>> cache_a = SharedRedisCache("redis://localhost:6379/0", ttl_seconds=300, similarity_threshold=0.92)
>>> cache_a.set("What is the capital of France?", "Paris is the capital of France.")
>>> cache_b = SharedRedisCache("redis://localhost:6379/0", ttl_seconds=300, similarity_threshold=0.92)
>>> cache_b.get("What is the capital of France?")
('Paris is the capital of France.', 1.0)
```

`cache_b` never called `.set()` — it read a value written by `cache_a`, proving state is shared
through Redis rather than held in-process.

### Redis CLI output

```bash
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
rl:cache:cfc5e751cb29

$ docker compose exec redis redis-cli HGETALL rl:cache:cfc5e751cb29
query
What is the capital of France?
response
Paris is the capital of France.
```

### In-memory vs Redis latency comparison (optional)

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | 238.6 | not separately load-tested | Redis adds a local-network round trip per cache lookup (~sub-ms on localhost) vs. in-process list scan; at this workload size (100 entries) the difference is negligible compared to the 180-260ms simulated provider latency. |
| latency_p95_ms | 318.44 | not separately load-tested | Same reasoning — dominated by provider latency, not cache backend. |

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Primary fails 100%, all traffic should fall back to backup | availability=0.98, fallback_success_rate=0.957, circuit_open_count=6, static_fallbacks=2 — backup absorbed almost all traffic, breaker cycled open/half-open on primary as expected | ✅ Pass |
| primary_flaky_50 | Primary fails 50%, circuit should oscillate between healthy calls and fallback | availability=1.0, fallback_success_rate=1.0, circuit_open_count=0 — with `failure_threshold=3` and only ~50% failure rate, 3-in-a-row is rare enough that the breaker rarely tripped; backup covered every failure that did occur | ✅ Pass |
| all_healthy | Both providers healthy (overridden to 2% fail rate), all traffic via primary, no circuit opens | availability=1.0, circuit_open_count=0, cache_hit_rate=0.67 | ✅ Pass |
| both_degraded (custom) | Primary 70% / backup 60% fail rate — expect static fallback responses and low availability once both breakers open | availability=0.0, static_fallbacks=100, circuit_open_count=2 — both providers failed enough to trip their breakers and every request landed on the static fallback message | ✅ Pass |

`both_degraded` is the custom scenario added beyond the 3 in the starter config
(`configs/default.yaml`), added to exercise the "everything is down" path and confirm the gateway
degrades gracefully (static message, no exception) instead of crashing.

## 8. Failure analysis

**Remaining weakness:** the combined-run SLIs in §3 look bad (79% availability, 40% fallback
success) purely because `both_degraded` is included in the same aggregate — a single badly-behaving
scenario can mask three healthy ones in a top-level dashboard. In a real production system, an
on-call engineer looking only at the blended availability number would over-react (or under-react,
if the blend happened to look fine) without per-scenario or per-provider breakdown.

**Proposed fix:** emit metrics with a `scenario` (or in production, a `route`/`provider`) label
dimension instead of one global scalar — e.g., a Prometheus counter/histogram per provider and
circuit state, so availability can be sliced by provider and by traffic segment. This is exactly
why `RunMetrics.scenarios` and per-provider `transition_log`s already exist in this codebase; the
next step is exporting them as labeled time series rather than only a single flattened JSON/CSV
row.

## 9. Next steps

1. Add cost-aware routing (stretch goal): once cumulative `estimated_cost` crosses a budget
   threshold, route to the cheaper `backup` provider or cache-only mode instead of always trying
   `primary` first.
2. Store circuit breaker counters in Redis (`INCR`/`EXPIRE`) so breaker state is shared across
   gateway replicas the same way the cache already is — right now each replica's breaker is
   independent, so one replica could still be hammering a dead provider while another has already
   tripped open.
3. Export metrics as labeled Prometheus series (per scenario/provider) instead of one flattened
   JSON, per the failure analysis above, so aggregate dashboards don't hide localized failures.
