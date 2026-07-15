# DEVSYM-373 — [BE] Superuser roles + cross-PM access policy + fan-out

> Session 2026-07-10. Branch `feat/devsym-373-superuser-roles-and-cross-pm-access-policy` off `dev`.

## What was done
Built the pure SA-domain machinery on top of the DEVSYM-399 foundation: role catalog,
`active` scope, access policy, and the sequential cross-PM fan-out. No table, no endpoint,
no auth change — exactly the refined scope. Produces the `fanOut(Role, callable): array<int
rm_location_id, array>` contract DEVSYM-374 consumes.

## Files touched
- `app/Modules/Superuser/Role.php` (new) — `enum Role: string`, five cases, flat catalog, no methods.
- `app/Modules/Superuser/Services/SuperuserAccessPolicy.php` (new) — `reachableConnections(Role): Collection<int,PmConnection>` → all active.
- `app/Modules/Superuser/Services/CrossPmFanout.php` (new) — `fanOut(Role, callable): array` keyed by `rm_location_id`, sequential, no merge/cache.
- `app/Models/PmConnection.php` (+14) — `const STATUS_ACTIVE` + `scopeActive()`. Nothing else touched.
- `tests/Unit/Modules/Superuser/{RoleTest, Services/SuperuserAccessPolicyTest, Services/CrossPmFanoutTest}.php` (new) — 4 tests, 8 assertions.

## Decisions made
- **No ServiceProvider.** No bindings/routes needed; deps auto-resolve via the container. YAGNI.
- **`Role` is a flat enum with no methods.** Unlike `TaskStatus::internalId()` (which exists for
  arithmetic next-state), `Role` has no order/transition, so no `match` method and no `label()` —
  a `label()` would be a caller-less stub in 373; display name is DEVSYM-376's when it seeds.
- **`STATUS_ACTIVE` const** instead of the raw `'active'` string (no magic constant).
- **PHPUnit `createMock` over Mockery** in `CrossPmFanoutTest` — see correction/RULES.md.
- **`Role` param kept but unused** in the policy — the forward-compat per-role scoping seam the
  ticket mandates, documented inline.

## Verification
- Module tests: 4 passed. Full suite: 22 failed / 252 passed — the 22 are pre-existing on `dev`
  (baseline confirmed 22 failed / 248 passed via stash), zero regressions.
- Pint clean, PHPStan level 5 clean (also enforced by the captainhook pre-commit hook, which is active).

## PR #139 review (jona872, CHANGES_REQUESTED) — addressed
- **Finding:** `CrossPmFanout::fanOut()` had no per-connection isolation — one connection whose
  `clientForConnection()` or `$operation()` throws (e.g. `RentManagerAuthenticationException` from
  `PmConnectionContext::decodeCredentials()` on a null/malformed `rm_credential_ref`) aborted the
  whole fan-out.
- **Fix (commit `fa29fb6`):** wrapped the loop body in `try/catch (\Throwable)`, sanitized
  `Log::warning` (exceptionClass + message only, no full Throwable — PII), skip the failed
  connection. Mirrors `OnboardingService::initTasks()`. Documented the "may return FEWER keys than
  reachableConnections()" contract on `fanOut()`'s `@return` for DEVSYM-374.
- **Tests added:** `test_fan_out_skips_a_failing_connection_and_keeps_the_rest` (throw in
  `clientForConnection`) + `test_fan_out_skips_a_connection_whose_operation_throws` (throw in
  `$operation`). Module suite: 6 passed. Pint + PHPStan clean.
- **Recorded BEFORE the fix (PR-review protocol):**
  - Correction file: `sessions/2026-07/BE/2026-07-10_19-00_correction-fanout-per-item-isolation.md`
  - RULES.md → RM Integration: "Any fan-out / loop over N independent RM targets must isolate each
    iteration in try/catch, log sanitized (class + message only, never the full Throwable — PII),
    skip the failed target, and accumulate partial results. Mirror `OnboardingService::initTasks()`."

## Open questions
- None for 373. Flag for 374: `EnsureRmTokenValid`'s expiry gate was validated only against
  `UserSessionContext`; 373 is the first `clientForConnection` caller but adds no gated route, so
  the gate is not exercised here — 374 (which owns the endpoint) must re-check it (RULES.md / DEVSYM-404).
