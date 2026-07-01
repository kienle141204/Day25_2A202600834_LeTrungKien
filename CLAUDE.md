# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Day 10 lab for production-style reliability engineering of an LLM agent gateway. The repo is a starter skeleton — the student implements reliability primitives (circuit breaker, semantic cache, gateway routing, chaos testing) from scratch until all 25 failing tests pass.

No real LLM API keys are needed. `FakeLLMProvider` simulates everything locally.

## Commands

```bash
# Install (use conda env ai-lab or a venv)
pip install -e ".[dev]"

# Run all tests
make test
# Run a single test file
pytest tests/test_circuit_breaker.py -v

# Lint / type check
make lint        # ruff
make typecheck   # mypy --strict

# Chaos simulation (needs gateway + circuit breaker + cache done)
make run-chaos   # produces reports/metrics.json

# Generate markdown report from metrics
make report

# Redis (required for SharedRedisCache tests)
make docker-up   # starts Redis via docker compose
make docker-down

# Clean generated artifacts
make clean
```

## Architecture

```
configs/default.yaml          ← provider fail rates, CB thresholds, cache config, chaos scenarios
src/reliability_lab/
  config.py          ← Pydantic models for YAML config (no changes needed)
  providers.py       ← FakeLLMProvider — simulates latency/failures/cost (no changes needed)
  circuit_breaker.py ← TODO: 3-state machine (CLOSED → OPEN → HALF_OPEN → CLOSED)
  cache.py           ← TODO: ResponseCache (in-memory, n-gram cosine) + SharedRedisCache (Redis)
  gateway.py         ← TODO: ReliabilityGateway.complete() — cache → breaker → fallback chain
  chaos.py           ← TODO: run_scenario(), calculate_recovery_time_ms()
  metrics.py         ← TODO: write_csv() export
scripts/
  run_chaos.py       ← CLI entry for chaos simulation
  generate_report.py ← generates final_report.md from metrics.json
tests/
  test_circuit_breaker.py   ← 11 tests (target: all pass)
  test_cache.py             ← 9 tests (target: all pass)
  test_gateway_contract.py  ← 4 tests (target: all pass)
  test_todo_requirements.py ← 7 xfail markers — should become unexpected PASS when done
  test_redis_cache.py       ← 6 tests — auto-skipped if Redis not running
  test_config.py            ← 2 tests (already passing)
  test_metrics.py           ← 2 tests (already passing)
```

## Implementation order

**1. Circuit breaker** (`circuit_breaker.py`) — implement 4 methods:
- `allow_request()`: CLOSED → allow; HALF_OPEN → allow; OPEN → check timeout, transition to HALF_OPEN if elapsed
- `call(fn, ...)`: check `allow_request()` → raise `CircuitOpenError` if denied; wrap call with `record_success`/`record_failure`
- `record_success()`: reset failure_count, increment success_count; if HALF_OPEN and success_count ≥ threshold → CLOSED
- `record_failure()`: increment failure_count, reset success_count; **HALF_OPEN** → re-open with reason `"probe_failure"` **elif** failure_count ≥ threshold → open with reason `"failure_threshold_reached"` (must be `if/elif`, not `or`)

**2. Cache** (`cache.py`) — `ResponseCache`:
- `similarity(a, b)`: cosine similarity over words + character 3-grams using `Counter` vectors (not Jaccard)
- `get(query)`: privacy check → evict expired → find best match → false-hit guard → return `(value, score)`
- `set(query, value, metadata)`: privacy check → append `CacheEntry`
- Add `self.false_hit_log: list[dict[str, object]] = []` in `__init__`

**3. Gateway** (`gateway.py`) — `complete(prompt)`:
- Cache check → provider chain (primary/fallback) with circuit breakers → static fallback

**4. Chaos + Metrics** (`chaos.py`, `metrics.py`):
- `run_scenario()`: loop N requests, collect latency/cost/cache/error stats
- `calculate_recovery_time_ms()`: scan transition_log for open→closed pairs, average delta
- `write_csv()`: flatten `to_report_dict()`, write single-row CSV

**5. Redis cache** (`cache.py`) — `SharedRedisCache.get()`/`set()`: exact match via hash key, similarity scan via `scan_iter`, same privacy/false-hit guards as in-memory cache

## Key design constraints

- `record_failure()` HALF_OPEN and threshold branches **must** use `if/elif` — they produce different `reason` strings logged in `transition_log`
- `similarity()` must use n-gram cosine, not Jaccard — graded test explicitly checks for n-gram behavior
- Privacy patterns (`PRIVACY_PATTERNS`) and false-hit detection (`_looks_like_false_hit`) are module-level helpers in `cache.py` — use them in both `ResponseCache` and `SharedRedisCache`
- Use `time.monotonic()` for timeout comparisons in circuit breaker; `time.time()` for `transition_log` timestamps and cache TTL
