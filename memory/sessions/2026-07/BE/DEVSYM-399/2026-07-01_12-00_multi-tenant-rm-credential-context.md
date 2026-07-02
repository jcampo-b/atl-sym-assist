# DEVSYM-399 — Multi-tenant foundation: request-resolved RM credential context

## What was done

Introduced a Strategy pattern (`RentManagerContext`) that decouples credential
resolution from `Auth::user()` in `RentManagerHttpClient`. The client now
consumes an injected context instead of reading directly from the Auth facade.
Added `pm_connections` table, `PmConnection` model, `UserSessionContext`
(verbatim move of existing logic), `PmConnectionContext` (Mode A partner_token
+ Mode B user_session), `RentManagerContextResolver`, and wired the default
binding in `AppServiceProvider`. All 16 pre-existing tests in
`RentManagerHttpClientTest` pass unchanged (constructor call updated
`new RentManagerHttpClient($authService)` →
`new RentManagerHttpClient(new UserSessionContext($authService))`).
9 new tests added in `PmConnectionContextTest`.

## Files touched

- `database/migrations/2026_07_01_000001_create_pm_connections_table.php` — new
- `app/Models/PmConnection.php` — new
- `app/Modules/Integrations/RentManager/Services/Context/RentManagerContext.php` — new (interface)
- `app/Modules/Integrations/RentManager/Services/Context/UserSessionContext.php` — new
- `app/Modules/Integrations/RentManager/Services/Context/PmConnectionContext.php` — new
- `app/Modules/Integrations/RentManager/Services/Context/RentManagerContextResolver.php` — new
- `app/Modules/Integrations/RentManager/Services/RentManagerHttpClient.php` — refactored
- `app/Providers/AppServiceProvider.php` — added RentManagerContext binding
- `config/rent_manager.php` — added credential_mode and partner_token keys
- `tests/Unit/Modules/Integrations/RentManager/Services/RentManagerHttpClientTest.php` — constructor updated
- `tests/Unit/Modules/Integrations/RentManager/Services/Context/PmConnectionContextTest.php` — new

## Decisions made

- Interface method is `handleUnauthorized(): ?string` only — `refreshTokenAfterUnauthorized(): bool`
  was rejected as redundant (the return value of null already communicates "no retry").
- All context classes live under `Services/Context/` (singular) — no separate `Contracts/` dir.
- `X-RM12Api-LocationID` header is NOT added in `UserSessionContext` — user_session path
  stays byte-identical to production. Only `PmConnectionContext` sends it.
- `PmConnectionContext` caches the PM token for 10 minutes (`CACHE_TTL_MINUTES = 10`).
  RM invalidates tokens after 15 min of inactivity; 10 min is a safe conservative ceiling.
- No auto-fallback from partner_token mode to user_session mode — `credential_mode` is
  config-driven and explicit; no silent degradation.

## Out-of-scope finding — preexisting TTL discrepancy (DEVSYM-400)

**`RentManagerAuthService::authenticate()`** (line 66):
```php
Cache::put(self::CACHE_KEY, $token, now()->addHours(23));
```

**`UserSessionContext::reauthenticateUser()`** (line 76, moved verbatim from
original `RentManagerHttpClient`):
```php
'rm_token_expires_at' => now()->addHours(23),
```

Both use a 23h TTL. RM invalidates tokens after **15 minutes of inactivity**
(official RM API docs). This means a user who is idle for >15 min will get a
401 on their next request even though the app considers their token valid for
another ~22h. The `handleUnauthorized()` retry is the actual safety net here,
but the 23h TTL is misleading and wastes cache space.

**Not fixed in DEVSYM-399** — this PR is behaviour-preserving; touching these
values would change the system token and per-user token behaviour for all
existing callers. Tracked as **DEVSYM-400**.

## Open questions

- Partner Token scope: one token multi-Location, or one per Location?
  (DEVSYM-393 — blocked on RM providing the token)
- Egress IP whitelisting for Mode A (DEVSYM-394)
