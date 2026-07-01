# Day 10 Reliability Final Report

## 1. Architecture Summary

```text
User Request
    |
    v
[ReliabilityGateway]
    |
    +--> [ResponseCache / SharedRedisCache] -- hit --> return cached response
    |
    v miss
[CircuitBreaker: primary] -- closed/half-open --> primary provider
    |
    v fail/open
[CircuitBreaker: backup] ---------------------> backup provider
    |
    v fail/open
[Static fallback response]
```

The gateway checks cache first to reduce latency and cost. On a miss, each provider call is protected by a circuit breaker. If the primary provider fails or its circuit is open, traffic falls back to the backup provider. If all providers fail, the gateway returns a static degraded response.

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Opens the circuit after repeated failures while tolerating occasional provider noise. |
| reset_timeout_seconds | 2 | Lets the provider cool down before a half-open probe. |
| success_threshold | 1 | One successful probe is enough for this local fake-provider lab. |
| cache backend | memory | Default run uses local memory; Redis backend is tested separately. |
| cache TTL | 300 | Keeps repeated lab queries reusable without keeping stale data too long. |
| similarity_threshold | 0.92 | Conservative threshold to reduce false semantic hits. |
| load_test requests | 100 per scenario | Three scenarios produce 300 total requests. |

## 3. SLO Definitions

| SLI | SLO Target | Actual Value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.00% | Yes |
| Latency P95 | < 2500 ms | 318.23 ms | Yes |
| Fallback success rate | >= 95% | 95.59% | Yes |
| Cache hit rate | >= 10% | 63.33% | Yes |
| Recovery time | < 5000 ms | N/A | No recovery event recorded in cached run |

## 4. Metrics

| Metric | Value |
|---|---:|
| total_requests | 300 |
| availability | 0.99 |
| error_rate | 0.01 |
| latency_p50_ms | 271.6 |
| latency_p95_ms | 318.23 |
| latency_p99_ms | 319.62 |
| fallback_success_rate | 0.9559 |
| cache_hit_rate | 0.6333 |
| circuit_open_count | 8 |
| recovery_time_ms | null |
| estimated_cost | 0.046704 |
| estimated_cost_saved | 0.19 |

## 5. Cache Comparison

| Metric | Without Cache | With Cache | Delta |
|---|---:|---:|---:|
| availability | 0.9733 | 0.99 | +0.0167 |
| latency_p50_ms | 275.48 | 271.6 | -3.88 ms |
| latency_p95_ms | 314.86 | 318.23 | +3.37 ms |
| estimated_cost | 0.122692 | 0.046704 | -0.075988 |
| cache_hit_rate | 0.0 | 0.6333 | +0.6333 |
| circuit_open_count | 22 | 8 | -14 |

Cache significantly reduced provider calls and estimated cost. The P95 latency difference is small because cached responses are mixed with provider calls, and provider latency dominates non-cache requests.

## 6. Redis Shared Cache

In-memory cache is insufficient for production multi-instance deployments because each gateway process has its own local cache. A user routed to another instance would miss entries already computed elsewhere.

`SharedRedisCache` solves this by storing cache entries in Redis hashes with TTL. Multiple gateway instances using the same Redis URL and prefix can share responses.

Evidence of shared state:

```text
instance_2_value=shared report response
score=1.0
keys=['rl:evidence:853c11f9f307']
```

Redis test result:

```text
pytest tests/test_redis_cache.py -q
6 passed
```

## 7. Chaos Scenarios

| Scenario | Expected Behavior | Observed Behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Primary fails; traffic falls back to backup or static fallback. | Scenario completed successfully. | Pass |
| primary_flaky_50 | Circuit opens during repeated primary failures; backup handles many requests. | Scenario completed successfully. | Pass |
| all_healthy | Primary should handle most non-cached requests. | Scenario completed successfully. | Pass |

## 8. Failure Analysis

One remaining weakness is that circuit breaker state is still process-local. In a real multi-instance deployment, one gateway instance could open its circuit while another keeps sending traffic to the unhealthy provider.

Before production, I would store circuit breaker counters and state in Redis with atomic operations, or use a shared health signal per provider. I would also add per-route SLO alerts and record fallback reasons per provider for easier incident debugging.

## 9. Final Validation

```text
pytest -q
35 passed, 7 xpassed
```

Generated artifacts:

- `reports/metrics.json`
- `reports/metrics_no_cache.json`
- `reports/metrics.csv`
- `reports/final_report.md`
