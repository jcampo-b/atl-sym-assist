# DEVSYM-305 — Auth token TTL / refresh diagnosis

## What was done
Investigated the dual-auth system to answer the ticket's two checklist items
("Is the main user refresh_token working?" / "Is the Sanctum refresh_token
working?") and increased the Sanctum token TTL default.

Key finding: **there is NO refresh token anywhere in this codebase** — neither
for the main user nor for Sanctum. The ticket's premise was wrong: nothing is
"broken", the feature simply does not exist. Sanctum personal access tokens are
not refreshable by design. The only way to obtain a new token is `POST
/api/auth/login`.

Decision (user): just increase the TTL as a stopgap. Refresh endpoint NOT built
this ticket.

## Auth system map (reference for future auth issues)

### Sanctum (SA users)
- TTL config: `config/sanctum.php` line 50 — `env('SANCTUM_TOKEN_EXPIRY', ...)`.
  Was `60 * 24 * 7` (7 days). Now `60 * 24 * 30` (30 days). Env var overrides.
- Token issued in `app/Modules/Auth/Services/AuthService.php:43` —
  `$user->createToken('api')` — no abilities (defaults to `['*']`), no per-token
  `expiresAt`, so the global config governs expiry.
- Routes (`app/Modules/Auth/routes.php`): only `POST /api/auth/login`
  (throttle 10/1), `POST /api/auth/logout` (auth:sanctum), `GET /api/auth/me`
  (auth:sanctum). **No `refresh` route. No RefreshController. No refresh method.**
- `config/auth.php` defines only the `web` guard. Sanctum works via the
  `auth:sanctum` middleware checking the bearer token against
  `personal_access_tokens`.

### RentManager token (RM layer)
- System token: `RentManagerAuthService` (app/Modules/Integrations/RentManager/
  Services/RentManagerAuthService.php).
  - Cache key: `rent_manager:api_token` (const, line 11).
  - TTL: **23h HARDCODED** — `Cache::put(..., now()->addHours(23))` line 77.
    Not config-driven. Magic number.
  - No proactive refresh: cache-aside only. Re-fetches on cache miss.
  - 401 handling: `RentManagerHttpClient::sendWithRetryRaw()` (~line 163) clears
    the cache on a 401 and retries once → implicit re-auth on RM-side expiry.
- Per-user RM token: stored on `users.rm_token` with
  `rm_token_expires_at = now()->addHours(23)` set at login (AuthService.php:39).
  - **KNOWN BUG (latent):** in `RentManagerHttpClient::client()` (~line 23), once
    the per-user token expires it SILENTLY falls back to the system 4MK token
    instead of re-authenticating as the user. Wrong audit attribution / perms
    after 23h. Spun out to its own ticket (see Open questions).

## Files touched
- `SymAssist-Backend/config/sanctum.php` line 50 — default `60*24*7` → `60*24*30`.

## Decisions made
- Sanctum TTL default raised to 30 days (43200 min). Env var
  `SANCTUM_TOKEN_EXPIRY` still overrides per environment.
- The committed change lives in `config/sanctum.php` (not `.env.example`) because
  the `.env*` directory is permission-blocked in this environment — could not
  read or write it. Document `SANCTUM_TOKEN_EXPIRY=43200` in `.env`/`.env.example`
  manually.
- RM per-user token silent-fallback bug is OUT OF SCOPE for DEVSYM-305 — separate
  ticket (user's decision).
- No refresh endpoint built. "Increase TTL" is a stopgap; it does not address the
  real gap (no token rotation, tokens carry `['*']` abilities, longer TTL = longer
  blast radius for a leaked token).

## Open questions / follow-ups
- Separate ticket: fix silent per-user RM token degradation to system token after
  23h in `RentManagerHttpClient::client()` — re-auth the user or fail explicitly.
- Future: if real token rotation is wanted, build `POST /api/auth/refresh` that
  revokes the current PAT and issues a new one. Not done here.
- The 23h RM TTL is a hardcoded magic number — candidate to move to config if it
  ever needs tuning per environment.
