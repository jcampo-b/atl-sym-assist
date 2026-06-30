# DEVSYM-306 — Eliminate duplicate compute() calls in PropertyPerformanceService

## What was done
Added a request-scoped in-memory cache to `PropertyPerformanceService::compute()`.
Both the early-return path (empty properties) and the normal return path now write
to `$this->cache[$cacheKey]`. `computeSummary()` gets a free cache hit when called
after `compute()` on the same instance.

## Files touched

### Modified
- `app/Modules/Dashboard/Services/PropertyPerformanceService.php`
  — added `private array $cache = []`
  — added `makeCacheKey()` private helper (md5 of sorted params)
  — cache guard at top of `compute()`, both return paths assign to cache

### Created
- `tests/Unit/Modules/Dashboard/Services/PropertyPerformanceServiceTest.php`
  — 4 unit tests: single call, same-params cache hit, different-params cache miss,
    computeSummary() alone hits RM once

## Decisions made
- Cache key: `md5(json_encode([from, to, sorted(propertyIds), sorted(rentChargeTypeIds), sorted(glAccountIds)]))`
  Arrays sorted before hashing to prevent false misses on `[2,1]` vs `[1,2]`.
- Both early-return (empty properties) and normal return must write to cache —
  the early return exits before the normal assignment and would bypass the cache.
- In-memory only. Request-scoped by Laravel container (fresh instance per request).
  No Redis, no persistent store.
- No other files changed. Controller, computeSummary(), routes, FE contracts untouched.

## Commits
- `fix[DEVSYM-306]: add request-scoped cache to PropertyPerformanceService::compute()`
- `test[DEVSYM-306]: unit tests for PropertyPerformanceService compute cache`

## Open questions
- PR not yet opened. Awaiting user go-ahead.
