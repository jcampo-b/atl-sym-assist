# PR Review — fix/cast-int-redis-session (PR #314)

## Review opened
- Timestamp: 2026-08-04
- Repo: BE (SymAssist-Backend)
- Branch: `fix/cast-int-redis-session`
- Base branch: `dev`
- Issue: none (no Linear card — branch name used as identifier). PR #314, author `jona872`.

## Dev's stated focus
From the PR body:
1. `PmConnectionContext` re-authenticated against RRM once per tenant/owner/vendor lookup even within a single script execution, because nothing short-circuited repeat calls when the persistent Cache store was unreachable. Adds a per-process static token cache checked before `Cache`, so a PM connection authenticates at most once per run.
2. Cast the cache TTL to `int` before `Carbon::addMinutes()` — env vars are always strings, so setting `RM_TOKEN_CACHE_TTL_MINUTES` crashed every token fetch with a Carbon `TypeError`.
3. Baseline `SYSTEM_PROMPT_V3` (VoiceAIService) as an intentionally unused dormant constant, same as its `V2_LEGACY` sibling already is.

Notes from dev: "No tests endpoint, no linear card. Just funded using voice caller."

## Files in scope
```
app/Modules/Integrations/RentManager/Services/Context/PmConnectionContext.php
phpstan-baseline.neon
```

## Verification run (this review)
- `docker exec symassist-backend-app-1 php artisan test --filter PmConnectionContextTest`
  - on `origin/dev`: **17 passed** (48 assertions)
  - on `fix/cast-int-redis-session`: **6 failed, 11 passed**
- Full suite: `dev` 42 pre-existing failures (env/flaky) → branch 48. The 6 deterministic new failures are all in `PmConnectionContextTest`.
- `vendor/bin/pint --test` → PASS. `vendor/bin/phpstan analyse` → No errors.
- Carbon 3.13.1 empirically confirmed in-container:
  - `addMinutes("15")` → `TypeError: Carbon\Carbon::rawAddUnit(): Argument #3 ($value) must be of type int|float, string given` (so the dev's diagnosis is correct — **any** string crashes, not only a non-numeric one)
  - `addMinutes("")` and `addMinutes("abc")` → no error, `+0 minutes`
- `Illuminate\Cache\Repository::put()` confirmed: `if ($seconds <= 0) { return $this->forget($key); }`
- `run-regression.php` does not exist anywhere in the repo (`fd -HI regression`, and no git history under `scripts/*regression*`).

---

## Findings

### Blockers

---

**File:** `app/Modules/Integrations/RentManager/Services/Context/PmConnectionContext.php`
**Code snippet:**
```php
    private function resolveUserSessionToken(): string
    {
        $connectionId = $this->pmConnection->id;

        if (isset(self::$memoryTokens[$connectionId])) {
            return self::$memoryTokens[$connectionId];
        }
```
**Severity:** `blocker`
**What's wrong:** The static `$memoryTokens` array is never reset between tests, so a token memoized in one test leaks into every later test in the same PHPUnit process — 6 tests in `PmConnectionContextTest` fail on this branch and pass on `dev`.
**Why:** `code-review/SKILL.md` → Tests: "Do all existing tests still pass? (`php artisan test`)". Verified: `--filter PmConnectionContextTest` gives 17 passed on `origin/dev` and 6 failed / 11 passed on this branch. The suite's `setUp()` calls `Cache::flush()`, which cannot reach static class state. All the fixtures use `id: 1` / `id: 2`, so `'pm-token'` memoized by `test_http_client_sends_location_id_header_for_pm_context` is returned to every later `id: 1` test, and `authenticateUser()` is then never called (`expects($this->once())` sees 0 calls; `assertSame('token-72', ...)` gets `'pm-token'`).
Failing tests: `user session mode authenticates with pm credentials`, `user session mode caches pm token across requests`, `user session mode reauthenticates after 401`, `without force partner token default still honours credential mode`, `two pm connections use separate token caches`, `two pm connections with the same rm location id do not collide`.
**Suggested fix:** Do not hold the memo in class-static state. Move it to a small process-scoped collaborator bound as a container singleton (e.g. `PmTokenMemo` injected into the context): the container is rebuilt per test so the memo resets for free, while a single CLI run or request keeps one instance and gets exactly the de-duplication the PR wants. If the static array is kept instead, it needs an explicit reset hook invoked from `TestCase::setUp()` — and that hook is test-only scaffolding in production code, which is why the injected-memo option is preferable. Either way the 6 failing tests must be green, plus a new test asserting two `PmConnectionContext` instances for the same connection authenticate only once.
**Comment for the dev:** This breaks 6 existing tests in `PmConnectionContextTest` — they pass on `dev` and fail here (`php artisan test --filter PmConnectionContextTest`). The static `$memoryTokens` survives across tests in the same PHPUnit process, and `setUp()`'s `Cache::flush()` can't clear static class state, so the token memoized for `id: 1` in an earlier test is handed to every later one and `authenticateUser()` never gets called. Could we move the memo into a container-bound singleton instead of a `static` property? The container is rebuilt per test, so it resets for free, and a single CLI run/request still gets the de-duplication you're after.

---

**File:** `app/Modules/Integrations/RentManager/Services/Context/PmConnectionContext.php`
**Code snippet:**
```php
        Cache::put($cacheKey, $token, now()->addMinutes((int) config('rent_manager.token_cache_ttl_minutes')));
```
**Severity:** `blocker`
**What's wrong:** The cast is applied at one call site, but four other call sites still pass the raw config value into `addMinutes()` and keep the exact same `TypeError`.
**Why:** `RULES.md` → "RM token cache TTL": `config('rent_manager.token_cache_ttl_minutes')` is the **single source of truth**, and the rule exists precisely because the original bug came from the same value being handled in four separate places. Still unfixed with `RM_TOKEN_CACHE_TTL_MINUTES` set in env:
- `app/Modules/Integrations/RentManager/Services/RentManagerAuthService.php:83` — `Cache::put(self::CACHE_KEY, $token, now()->addMinutes(config('rent_manager.token_cache_ttl_minutes')));`
- `app/Modules/Integrations/RentManager/Services/Context/UserSessionContext.php:179` — `'rm_token_expires_at' => now()->addMinutes(config('rent_manager.token_cache_ttl_minutes')),`
- `app/Modules/Auth/Services/AuthService.php:105` — same expression
- `app/Modules/Onboarding/Services/OnboardingActivationService.php:192` — same expression

Confirmed against Carbon 3.13.1 in-container: `addMinutes("15")` throws `TypeError: Carbon\Carbon::rawAddUnit(): Argument #3 ($value) must be of type int|float, string given`. So **every** string crashes, including numeric ones — meaning login (`AuthService::login()`) and the system-token path break the moment that env var is set, whatever this PR does.
**Suggested fix:** Cast once at the source, in `config/rent_manager.php:45`:
`'token_cache_ttl_minutes' => env('RM_TOKEN_CACHE_TTL_MINUTES', 10),` → cast to a positive int there, and drop the per-call-site `(int)`. One line fixes all five consumers, keeps the single-source-of-truth rule intact, and survives `config:cache`.
**Comment for the dev:** Great catch on the root cause — I confirmed in-container that Carbon 3.13.1 throws on *any* string, numeric included. But the cast at this call site only rescues `PmConnectionContext`; `RentManagerAuthService:83`, `UserSessionContext:179`, `AuthService:105` and `OnboardingActivationService:192` all pass the same config value straight into `addMinutes()`, so login and the system-token path still crash with `RM_TOKEN_CACHE_TTL_MINUTES` set. `RULES.md` ("RM token cache TTL") makes this config key the single source of truth for exactly this reason — could we cast in `config/rent_manager.php:45` instead and drop the local cast? One line, all five consumers fixed.

---

### Should-fix

---

**File:** `app/Modules/Integrations/RentManager/Services/Context/PmConnectionContext.php`
**Code snippet:**
```php
        Cache::put($cacheKey, $token, now()->addMinutes((int) config('rent_manager.token_cache_ttl_minutes')));
        self::$memoryTokens[$connectionId] = $token;
```
**Severity:** `should-fix`
**What's wrong:** `(int)` turns a blank or non-numeric env value into `0`, and a `0`-minute TTL makes `Cache::put()` delete the key instead of storing it — the token is then silently never persisted.
**Why:** `Illuminate\Cache\Repository::put()` (verified in vendor): `$seconds = $this->getSeconds($ttl); if ($seconds <= 0) { return $this->forget($key); }`. `RM_TOKEN_CACHE_TTL_MINUTES=` (blank) returns `''` from `env()`, and `(int) '' === 0`; same for a typo'd value. So the cast converts a loud `TypeError` into silent cache-bypass, and the new in-memory layer then hides the symptom for the rest of the process. This is the failure mode `RULES.md` → "Code quality" calls out as no silent degradation: a fallback that fires because the "impossible" case happened must be diagnosable, not invisible.
**Suggested fix:** Fold this into the config-level fix and guarantee a positive value, e.g. `max(1, (int) env('RM_TOKEN_CACHE_TTL_MINUTES', 10))` in `config/rent_manager.php` — with a comment stating the RM 15-min inactivity ceiling, so it can never silently resolve to 0.
**Comment for the dev:** One consequence of the bare `(int)`: a blank or typo'd `RM_TOKEN_CACHE_TTL_MINUTES` becomes `0`, and `Cache::put()` with `$seconds <= 0` calls `forget()` instead of storing (see `Illuminate\Cache\Repository::put()`). The token then never persists and the new memory layer hides it for the rest of the process. Could we clamp to a positive value at the config level, e.g. `max(1, (int) env('RM_TOKEN_CACHE_TTL_MINUTES', 10))`?

---

**File:** `app/Modules/Integrations/RentManager/Services/Context/PmConnectionContext.php`
**Code snippet:**
```php
    /**
     * Per-process token cache, checked before the persistent Cache store.
     * Guarantees a single PM connection only ever authenticates once within
     * one script/request execution (e.g. run-regression.php resolving a
     * caller's identity across tenant/owner/vendor lookups), regardless of
     * whether the configured Cache store is reachable.
     *
     * @var array<int, string>
     */
    private static array $memoryTokens = [];
```
**Severity:** `should-fix`
**What's wrong:** The memory layer has no expiry at all, and the docblock's "regardless of whether the configured Cache store is reachable" claim is not actually achieved by the code.
**Why:** Two separate problems in one comment.
(a) `RULES.md` → "RM token cache TTL": "Never cache an RM token with a TTL longer than ~10 min" — RM invalidates tokens after 15 min of inactivity. `$memoryTokens` has an unbounded lifetime tied to the process. Under `php-fpm` that is one request, so today's blast radius is small, but `QUEUE_CONNECTION` defaults to `database` and there are five `ShouldQueue` jobs, so a long-lived `queue:work` (or any CLI run longer than 15 min) would serve an expired token from memory and rely entirely on the `401` → `handleUnauthorized()` retry as the safety net — the exact avoidable-401 pattern that rule was written to prevent.
(b) The resilience claim doesn't hold: `Cache::get($cacheKey)` runs two lines below the memory check, so an unreachable store throws there before the memory layer can help; and `self::$memoryTokens[$connectionId] = $token;` is written *after* `Cache::put(...)`, so a failing store write skips the memo entirely.
**Suggested fix:** Store `[token, expiresAt]` and honour the same `token_cache_ttl_minutes` ceiling in the memory layer, assign the memo *before* the `Cache::put()` call, and rewrite the docblock to state what the code actually guarantees (de-duplicated authentication within one process, bounded by the same TTL) rather than store-independence it does not provide.
**Comment for the dev:** Two things on this docblock. First, the memo has no expiry, while `RULES.md` ("RM token cache TTL") says never to cache an RM token beyond ~10 min because RM drops it after 15 min of inactivity — fine under php-fpm, but a `queue:work` worker or a long CLI run would keep serving a dead token and lean on the 401 retry. Second, the "regardless of whether the Cache store is reachable" part isn't what the code does: `Cache::get()` runs two lines below and would throw first, and the memo is assigned *after* `Cache::put()`, so a failed store write skips it. Could we give the memo the same TTL ceiling, move the assignment above `Cache::put()`, and reword the comment to what it really guarantees?

---

### Nits

---

**File:** `app/Modules/Integrations/RentManager/Services/Context/PmConnectionContext.php`
**Code snippet:**
```php
     * one script/request execution (e.g. run-regression.php resolving a
```
**Severity:** `nit`
**What's wrong:** The docblock's only concrete justification points at a file that does not exist in the repo.
**Why:** `fd -HI regression` finds no such script, and there is no history under `scripts/*regression*` (`scripts/` holds only `rent-manager-discover.sh` and `rent-manager-sanity.sh`). `code-review/SKILL.md` → Code clarity keeps comments that explain a non-obvious WHY; a reader cannot verify a WHY that lives in a local file outside the repo.
**Suggested fix:** Describe the scenario in terms of code that exists (e.g. a single CLI run resolving the same connection through several lookups) or reference the committed script if it is meant to land.
**Comment for the dev:** Small one — the comment cites `run-regression.php`, which isn't in the repo (only `rent-manager-discover.sh` / `rent-manager-sanity.sh` under `scripts/`). Could we describe the scenario generically, or commit the script if it belongs here?

---

**File:** `phpstan-baseline.neon`
**Code snippet:**
```
		-
			message: '#^Constant App\\Modules\\VoiceAI\\Services\\VoiceAIService\:\:SYSTEM_PROMPT_V3 is unused\.$#'
			identifier: classConstant.unused
			count: 1
			path: app/Modules/VoiceAI/Services/VoiceAIService.php
		-
```
**Severity:** `nit`
**What's wrong:** The entry was hand-inserted without the blank line that separates every other entry in this generated file.
**Why:** Every other block in `phpstan-baseline.neon` is separated by a blank line (see the `SYSTEM_PROMPT_V2_LEGACY` entry immediately above). The file is `--generate-baseline` output; a hand edit drifts from what the tool would emit and produces noise in the next regenerated diff. The rest of the entry is correct and justified — `SYSTEM_PROMPT_V3` is genuinely unreferenced (only `V1` at line 329 and `V2` at line 629 are used), and the `V2_LEGACY` precedent the PR cites really is already baselined. `phpstan analyse` is clean on this branch.
**Suggested fix:** Regenerate with `vendor/bin/phpstan analyse --generate-baseline` so the formatting matches, or add the missing blank line.
**Comment for the dev:** Tiny formatting one — the new baseline entry is missing the blank line that separates every other block, so it looks hand-added. `vendor/bin/phpstan analyse --generate-baseline` would normalize it. The entry itself is fine, and the `V2_LEGACY` precedent checks out.

---

## Verdict
**needs changes before merge** — 2 blockers, 2 should-fix, 2 nits.

The two diagnoses behind this PR are both real and correctly identified (I reproduced the Carbon `TypeError` in-container). The problems are in the shape of the fixes: the TTL cast is applied at one of five call sites when the config key is meant to be the single source of truth, and the per-process memo is class-static, which breaks 6 existing tests.

## SPACE log
- Time spent reviewing: 30 min
- Model: Opus 5 (claude-opus-5)
- Findings: 2 blockers, 2 should-fix, 2 nits
- Verdict: needs changes before merge
