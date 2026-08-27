# Day 25 Reliability Report

## 1. Architecture summary

The gateway routes every prompt through a cache-aside layer, then a per-provider circuit
breaker, then an ordered fallback chain, and finally a static degraded response if nothing
else succeeds.

```
User Request
    |
    v
[ReliabilityGateway.complete(prompt)]
    |
    v
[Cache.get(prompt)] --------------------------> HIT (score >= threshold, not a false hit)
    | MISS / no cache                                 |
    v                                                  v
[Circuit Breaker: primary] --skip if OPEN-->   return route="cache_hit:{score:.2f}"
    | call primary.complete()
    | success -> cache.set() -> route="primary"
    | ProviderError / CircuitOpenError -> record error, continue
    v
[Circuit Breaker: backup] --skip if OPEN-->
    | call backup.complete()
    | success -> cache.set() -> route="fallback"
    | ProviderError / CircuitOpenError -> record error, continue
    v
[Static fallback]
  route="static_fallback"
  text="The service is temporarily degraded. Please try again soon."
  error=<last provider error>
```

Each provider has its own independent `CircuitBreaker` (CLOSED → OPEN → HALF_OPEN → CLOSED),
so a failing `primary` does not affect calls routed to `backup`. The cache sits in front of
both breakers, so a cache hit never touches a provider or a breaker at all — it costs zero
latency and zero provider spend.

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | `primary` has a 25% baseline fail rate, so single unlucky failures are common. Requiring 3 consecutive failures avoids tripping the breaker on noise while still reacting within a handful of requests to a real outage. |
| reset_timeout_seconds | 2 | Short enough to let the lab's chaos runs (100 requests/scenario, ~0.2-0.3s latency each) actually observe multiple open→half_open→closed cycles instead of staying open for the whole run. Confirmed by measured `recovery_time_ms` landing at ~2.2-2.4s across all three runs, i.e. right at the timeout as expected. |
| success_threshold | 1 | A single successful probe closes the circuit immediately, favoring fast recovery over caution. Trade-off: a lucky probe can close the circuit right before another real failure (acceptable here since `record_failure()` re-opens instantly on any HALF_OPEN failure with reason `probe_failure`). |
| cache TTL (`ttl_seconds`) | 300s | Sample queries are FAQ/technical/dated content that does not change every few seconds; 5 minutes balances freshness against reducing repeat provider calls for the 20-query sample set that repeats often over 100 requests. |
| similarity_threshold | 0.92 | Set high on purpose. Character n-gram cosine similarity is very forgiving of small edits (e.g. "refund policy for 2024" vs "... for 2026" scores well above 0.9 despite meaning something different). A lower threshold (e.g. 0.85) would raise the pre-guardrail hit rate but push more near-duplicate, different-intent queries into the false-hit path; 0.92 plus the explicit year/number guardrail (`_looks_like_false_hit`) is the safer combination since a wrong cached answer is worse than a cache miss. |
| load_test requests | 100 per scenario (300 total) | Large enough to get a stable P95/P99 estimate and to let the circuit breaker cycle open/close multiple times under `primary_flaky_50`, small enough to run the full 3-scenario suite in a few seconds locally. |

## 3. SLO definitions

Baseline run: `configs/default.yaml` (cache enabled, backend=memory), see `reports/metrics.json`.

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 98.33% | ❌ No (0.67 pts short) |
| Latency P95 | < 2500 ms | 317.38 ms | ✅ Yes |
| Fallback success rate | >= 95% | 94.44% | ❌ No (0.56 pts short) |
| Cache hit rate | >= 10% | 61.33% | ✅ Yes |
| Recovery time | < 5000 ms | 2244.90 ms | ✅ Yes |

Availability and fallback success rate both miss their target by a small margin. Both
misses trace back to the same cause: `backup`'s own 5% baseline fail rate. When `primary`
is at 100% failure (`primary_timeout_100` scenario) every request depends on `backup`
succeeding, so `backup`'s 5% failure rate becomes the system's error floor for that
scenario — there is no third fallback to absorb it. See section 8 for the proposed fix.

## 4. Metrics

Full JSON: `reports/metrics.json` (memory cache backend, the configuration used for the
required deliverable).

| Metric | Value |
|---|---:|
| total_requests | 300 |
| availability | 0.9833 |
| error_rate | 0.0167 |
| latency_p50_ms | 283.51 |
| latency_p95_ms | 317.38 |
| latency_p99_ms | 320.2 |
| fallback_success_rate | 0.9444 |
| cache_hit_rate | 0.6133 |
| circuit_open_count | 10 |
| recovery_time_ms | 2244.8976039886475 |
| estimated_cost | 0.04467 |
| estimated_cost_saved | 0.184 |

## 5. Cache comparison

Two identical 300-request runs (3 scenarios × 100 requests), same provider fail rates and
circuit breaker config, only `cache.enabled` differs.

- With cache: `configs/default.yaml` → `reports/metrics.json`
- Without cache: `configs/no_cache.yaml` → `reports/metrics_nocache.json`

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 271.67 | 283.51 | +11.84 ms (not a meaningful cache effect — see note below) |
| latency_p95_ms | 315.52 | 317.38 | +1.86 ms (not a meaningful cache effect — see note below) |
| estimated_cost | 0.130008 | 0.04467 | **-0.0853 (-66%)** |
| cache_hit_rate | 0.0 | 0.6133 | +0.6133 |
| circuit_open_count | 18 | 10 | -8 (44% fewer trips) |

**Reading the numbers:** the cost drop (-66%) is the headline result — cache hits skip the
provider entirely, so 61% of requests cost nothing. The P50/P95 numbers are *not* diluted
by cache hits at all: `run_scenario()` only appends to `latencies_ms` when
`result.latency_ms > 0`, and a cache hit is built with `latency_ms=0.0`, so cache hits are
excluded from the latency sample by construction. That means the "with cache" percentiles
above are measured purely over the ~39% of requests that were genuine cache misses
(real provider calls) — the same population as "without cache", where every request is a
provider call — which is exactly why the two columns land so close together: they are
measuring the same kind of request, just fewer of them. The cache's actual latency benefit
(near-zero response time for 61% of traffic) is real but invisible in P50/P95 as currently
computed; it shows up correctly in `estimated_cost` and, more directly, in `availability`
(cached responses can never fail). A more complete percentile view would fold cache hits
into the latency sample as `0.0` explicitly rather than skip them — noted in section 9.
The more interesting side effect is `circuit_open_count`: with the cache
absorbing most repeat traffic, `primary` receives fewer real calls overall, so it trips its
breaker roughly half as often (10 vs 18) purely because it is asked to do less work — the
cache is indirectly protecting the circuit breaker from noise.

## 6. Redis shared cache

- Why in-memory cache is insufficient for multi-instance deployments: `ResponseCache`
  stores entries in a plain Python list (`self._entries`) that lives inside one process's
  memory. If the gateway is scaled to N instances/pods behind a load balancer, each
  instance has its own empty cache — the same question asked twice could hit two different
  instances and pay the provider cost twice, and TTL eviction / false-hit stats
  (`false_hit_log`) are also per-instance and never reconciled.
- How `SharedRedisCache` solves this: all instances point at the same Redis server. A
  cache entry is a Redis Hash (`{"query": ..., "response": ...}`) under key
  `rl:cache:{md5(query)[:12]}`, with `EXPIRE` handling TTL server-side instead of manual
  per-instance eviction. Because the data lives outside any single process, any instance
  that calls `get()` sees writes made by any other instance.

### Evidence of shared state

`tests/test_redis_cache.py::test_shared_state_across_instances` constructs two independent
`SharedRedisCache` Python objects (`c1`, `c2`) against the same `redis://localhost:6379/0`,
writes through `c1`, and reads through `c2`:

```
tests/test_redis_cache.py::test_shared_state_across_instances PASSED
```

`c2.get("shared query")` returned the exact value `c1.set(...)` wrote, which is only
possible if the state lives outside both Python processes' memory — i.e. in Redis.

### Redis CLI output

Ran the chaos suite against `configs/redis.yaml` (`cache.backend: redis`) after
`FLUSHDB`, then listed the keys:

```bash
$ docker compose exec redis redis-cli FLUSHDB
OK
$ python scripts/run_chaos.py --config configs/redis.yaml --out reports/metrics_redis.json
wrote reports/metrics_redis.json
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
 1) "rl:cache:d354658dc020"
 2) "rl:cache:b2a52f7dc795"
 3) "rl:cache:8baa2cfa11fa"
 4) "rl:cache:734852f3cf4a"
 5) "rl:cache:98332d0d1c9c"
 6) "rl:cache:9e413fd814eb"
 7) "rl:cache:0bc3b1acf73d"
 8) "rl:cache:095946136fea"
 9) "rl:cache:fff10da1c72c"
10) "rl:cache:844ef0143a5c"
11) "rl:cache:3dab98c0e49e"
12) "rl:cache:dacb2b833659"
```

12 distinct keys is consistent with the 20-query sample set (`data/sample_queries.jsonl`):
some queries are privacy-flagged and never reach Redis at all (`_is_uncacheable`), and
some near-duplicate queries collapse into the same key via the exact-match hash or get
served as a similarity hit before a new key would be written.

### In-memory vs Redis latency comparison

Same 300-request run, only the cache backend differs (`configs/default.yaml` = memory vs
`configs/redis.yaml` = redis):

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | 283.51 | 275.55 | Comparable — well within run-to-run noise from `FakeLLMProvider`'s random jitter |
| latency_p95_ms | 317.38 | 317.57 | Comparable |
| cache_hit_rate | 0.6133 | 0.6733 | Comparable magnitude; exact value differs because `random.choice` is unseeded (see section 9) |

**Caveat:** re-running chaos against Redis a second time without `FLUSHDB` first would
inflate `cache_hit_rate` because entries from the previous run are still live under the
300s TTL — this run was measured immediately after `FLUSHDB` to keep the comparison fair.

## 7. Chaos scenarios

All three scenarios ran as part of the same 300-request `reports/metrics.json` execution;
per-scenario pass/fail comes from `RunMetrics.scenarios` (see section 9 for the current
scenario-level pass criteria).

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | `primary` fails every call; the `primary` breaker should open almost immediately (3 consecutive failures) and nearly all traffic should succeed via `backup` fallback instead. | `primary`'s breaker opens within the first few requests and stays open (still fails on every HALF_OPEN probe, so it re-opens instead of recovering); traffic is served by `backup` or cache. Combined `fallback_success_rate` across all scenarios is 94.44%, limited by `backup`'s own 5% fail rate. | pass |
| primary_flaky_50 | `primary` fails half the time; its breaker should cycle OPEN → HALF_OPEN → CLOSED repeatedly instead of staying in one state ("oscillate"), producing both `primary` and `fallback` routes. | This scenario is the main source of `circuit_open_count` (10 opens combined across all scenarios) and of measurable `recovery_time_ms` (~2245ms, matching `reset_timeout_seconds=2`) — evidence the breaker is actually cycling rather than latching open. | pass |
| all_healthy | No provider overrides (`primary` at its baseline 25% fail rate, `backup` at 5%); most traffic should succeed via `primary`, cache hit rate should climb once queries repeat. | Provides the bulk of `cache_hit_rate` (61.33% overall) since queries repeat across the 100 requests in this scenario with no injected failures forcing a cold cache. | pass |

**Limitation:** `RunMetrics` combines all scenarios into one set of aggregate numbers in
`metrics.json` — there is no per-scenario latency/availability breakdown in the exported
file today, so "observed behavior" above is reasoned from the combined metrics plus the
known mechanics of each override, not from an isolated per-scenario metrics dump. Section 9
lists this as a concrete next step.

## 8. Failure analysis

**Weakness:** availability (98.33%) and fallback success rate (94.44%) both miss their SLO
targets, and the gap traces to a single point of failure: once `primary`'s circuit is open,
every request funnels through exactly one remaining path — `backup`. `backup` has its own
5% baseline fail rate and no circuit-protected fallback behind *it*, so during
`primary_timeout_100` the system's floor error rate is bounded below by `backup`'s fail
rate with no further safety net. This shows up directly in the numbers two ways: (1) the
Redis run (`metrics_redis.json`), which hit the *same* `primary_timeout_100` scenario, saw
fallback_success_rate dip to 89.55% — run-to-run variance in how many of those `backup`
calls happen to fail (unseeded randomness, see section 9); (2) the no-cache run
(`metrics_nocache.json`) shows `circuit_open_count` jump to 18 (vs 10 with cache), because
without the cache absorbing repeat queries, `primary` receives far more real traffic and
trips its breaker roughly twice as often — all of that extra traffic still has to be caught
by the same single `backup` path.

A second, related weakness: circuit breaker state lives in each `CircuitBreaker` instance's
Python memory (`failure_count`, `state`, `transition_log`), exactly like the in-memory
cache problem in section 6. If this gateway were scaled to multiple pods, each pod would
have to independently rediscover that `primary` is failing before opening its own breaker
— there is no shared "primary is down" signal, so the first N requests on every new pod
still get sent to a provider already known to be broken elsewhere.

**Proposed fix:** add a third, cheaper static/cached-only tier behind `backup` specifically
for when both `primary` and `backup` breakers are open simultaneously, so the system never
depends on a single provider's raw fail rate as its error floor. Separately, migrate
circuit breaker counters into Redis (`INCR`/`EXPIRE` per breaker key, as noted in the
stretch goals) so all instances share one state machine per provider instead of N
independent ones — this also gives the Redis outage story (`SharedRedisCache` has no
documented in-memory fallback today) a natural companion fix: if Redis is unreachable, fall
back to a local in-memory `CircuitBreaker`/`ResponseCache` instance rather than failing the
whole gateway.

## 9. Next steps

1. **Seed randomness for reproducibility.** `FakeLLMProvider` and `random.choice(queries)`
   in `run_scenario()` are unseeded, so `metrics.json` differs slightly run to run (compare
   `metrics.json` vs `metrics_redis.json` — same config intent, different exact numbers).
   Adding a `seed` field to `configs/default.yaml` and threading `random.Random(seed)`
   through `build_gateway`/`run_scenario` would let two runs be byte-for-byte comparable.
2. **Per-scenario metrics breakdown.** Export a `RunMetrics` per scenario name (not just
   the merged total) in `metrics.json`, so section 7's "observed behavior" can cite exact
   per-scenario availability/latency instead of reasoning from the combined numbers.
3. **Redis-backed circuit breaker state + graceful Redis degradation**, as described in
   section 8 — closes the multi-instance gap for both reliability primitives (cache and
   breaker), not just the cache.
4. **Include cache hits in the latency sample.** `run_scenario()` currently skips
   `latency_ms == 0` entries, which hides the cache's real latency win from P50/P95 (section
   5). Recording cache hits as `0.0` in `latencies_ms` instead of skipping them would make
   the percentile metrics reflect the cache's actual end-to-end latency benefit.
