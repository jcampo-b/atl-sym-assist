# DEVSYM-400 — Centralize RM token cache TTL

## What was done

Added `token_cache_ttl_minutes` to `config/rent_manager.php` (default 10 min,
`RM_TOKEN_CACHE_TTL_MINUTES` env override) as the single source of truth for
the RM API token cache TTL, replacing a 23h hardcoded value that exceeded
RM's real 15-min inactivity window.

Updated four consumers to read the config key instead of hardcoding
`now()->addHours(23)`:
- `RentManagerAuthService::authenticate()` (system/service-account token)
- `UserSessionContext::reauthenticateUser()` (per-user token)
- `PmConnectionContext::resolveUserSessionToken()` (removed its own
  `CACHE_TTL_MINUTES = 10` constant, now reads the same config key)
- `AuthService::login()` (Auth module, initial login persistence)

Added two characterization tests (Feathers-style — pin current behavior so
a future regression breaks a test instead of failing silently in prod):
- `UserSessionContextTest::test_reauthenticate_persists_expiry_from_configured_ttl_and_second_call_skips_reauth`
- `RentManagerAuthServiceTest::test_authenticate_caches_token_with_configured_ttl`

Both verified to actually fail when the TTL calculation is reverted to
`addHours(23)` (regression-tested before finalizing, then reverted).

## Files touched

- `config/rent_manager.php`
- `app/Modules/Auth/Services/AuthService.php`
- `app/Modules/Integrations/RentManager/Services/RentManagerAuthService.php`
- `app/Modules/Integrations/RentManager/Services/Context/UserSessionContext.php`
- `app/Modules/Integrations/RentManager/Services/Context/PmConnectionContext.php`
- `tests/Unit/Modules/Integrations/RentManager/Services/RentManagerAuthServiceTest.php` (new)
- `tests/Unit/Modules/Integrations/RentManager/Services/Context/UserSessionContextTest.php` (new)

## Decisions made

- **4th undocumented location included in scope.** The ticket named three
  hardcoded 23h locations; exploration found a 4th in `AuthService::login()`
  (Auth module, not RM Integrations) with the identical bug. Included it
  after confirming with Johnny, since the AC says "no hardcoded token TTL
  anywhere" — leaving it out would ship the fix with the same bug still live
  in the initial-login path.
- **`PmConnectionContext`'s own `CACHE_TTL_MINUTES = 10` constant removed**
  rather than kept in sync manually — single source of truth was the point
  of the ticket.
- **TTL assertions in tests use `Mockery::on()` against exact `Carbon`
  equality with frozen time (`Carbon::setTestNow()`)**, not real cache
  expiry — the test cache driver may be Redis in this Docker setup, and
  Redis TTL is real wall-clock time, immune to `Carbon::setTestNow()`. This
  is the pattern to reuse for any future TTL characterization test in this
  codebase.
- **`RentManagerAuthService::CACHE_KEY` is `private`** — tests hardcode the
  literal `'rent_manager:api_token'` with a comment noting it must stay in
  sync with the constant. No reflection or constant exposure added — out of
  scope for this ticket.
- Existing `addHours(20)`/`subMinute()` fixtures in `RentManagerHttpClientTest`
  confirmed to be relative "still valid" / "already expired" markers, not
  assertions tied to the old 23h/24h threshold — left untouched.
- `phpstan`/`pint` clean; full test suite green except the 22 pre-existing
  `DEV_AUTH_BYPASS` 401→200 failures already present on `dev` (confirmed via
  `git stash` comparison — identical count on both branches).

## Open questions

None. Ticket AC fully covered.
