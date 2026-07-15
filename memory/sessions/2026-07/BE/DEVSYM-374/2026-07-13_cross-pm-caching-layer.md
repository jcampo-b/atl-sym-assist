# DEVSYM-374 — [BE] Cross-PM data aggregation and caching layer

> Session 2026-07-13. Branch `feat/devsym-374-cross-pm-data-aggregation-and-caching-layer` off `dev`.

## What was done

Built the cache + aggregation layer for the superuser Portfolio view on top of
DEVSYM-373's `CrossPmFanout`. A new `GET /api/superuser/portfolio` endpoint
fans out across every active PM connection, fetches properties/units/tenants/
owners (slow tier) and open tickets (fast tier), tags every record with
`rm_location_id`, and caches each connection's result independently via
`Cache::flexible` (SWR) — same mechanism as `PropertyPerformanceService`/
`VacancyService`, but keyed by `rm_location_id` instead of `Auth::id()`.

## Files touched

- `app/Modules/Superuser/Services/PortfolioCacheService.php` (new) — orchestrates
  the fan-out + per-connection cache + tagging + counts.
- `app/Modules/Superuser/Controllers/PortfolioController.php` (new)
- `app/Modules/Superuser/Requests/PortfolioRequest.php` (new) — validates `refresh`.
- `app/Modules/Superuser/routes.php` (new) — `['api','auth:sanctum','rm_super_admin']`.
- `app/Modules/Superuser/SuperuserServiceProvider.php` (new) — loads routes.php.
- `bootstrap/providers.php` (+1) — registers the new provider.
- `app/Modules/Integrations/RentManager/Services/RentManagerHttpClient.php` (+9) —
  new public `locationId(): ?int` getter, delegating to `activeContext()->locationId()`.
- `app/Modules/Integrations/RentManager/Services/RentManagerUnitService.php` (+18) —
  new `getActiveUnits()`: embeds `Property`, drops units whose `Property.IsActive`
  isn't true (RM has no unit-level IsActive).
- `RESTClient/superuser-portfolio.http` (new).
- Tests: `PortfolioCacheServiceTest`, `PortfolioControllerTest` (new), plus
  additions to `RentManagerHttpClientTest` and `RentManagerUnitServiceTest`.

## Decisions made

- **Middleware group has no `rm_token.valid`.** Confirmed via the DEVSYM-376
  refinement note precedent (admin-guarded endpoints with no RM call on the
  caller's own session drop it). The superuser's own RM session token is never
  used — all RM calls ride per-`PmConnection` tokens via `PmConnectionContext`.
- **New `RentManagerHttpClient::locationId()` getter instead of changing
  `CrossPmFanout::fanOut()`'s signature.** Keeps DEVSYM-373 fully untouched;
  the cache-key closure derives `rm_location_id` from the client itself.
- **Pagination via a private `fetchAllPages()` helper** (added in review) reused
  by all five `fetch*`. It terminates on pages REQUESTED vs source `total`
  (`page * per_page >= total`), NOT on accumulated output count — critical
  because `getActiveUnits()` filters rows client-side, so an output-count vs
  `total` comparison would never converge and would spin to the `MAX_PAGES`
  (50) safety cap. Reads only the already-normalized `meta.pagination.total`/
  `per_page`, so it stays SA-layer clean. `meta.counts` reports all five
  resources (properties/units/tenants/owners/tickets).
- **Role placeholder (`CALLER_ROLE_PLACEHOLDER = Role::PortfolioManager`)** in
  `PortfolioCacheService`, explicitly commented as inert-until-DEVSYM-376:
  `SuperuserAccessPolicy::reachableConnections()` ignores `$role` today (every
  role reaches every active connection), so this placeholder has no behavioral
  effect yet. Must be replaced with the authenticated caller's real role once
  376/the auth ticket resolves it — flagging this explicitly so it isn't missed.
- **Deleted/stale filtering — Properties + Units only.** Properties use the
  existing `IsActive,eq,true` RM filter (mirrors `RentManagerDashboardService`).
  Units have no own IsActive; filtering by the embedded `Property.IsActive` was
  moved INTO `RentManagerUnitService::getActiveUnits()` (RM layer), not left in
  `PortfolioCacheService` — an early draft read raw `Property.IsActive` directly
  in the Superuser (SA) module, which is a cardinal-rule violation (PascalCase
  RM field outside `Integrations/RentManager/`), caught during self-review.
  **Tenants and Owners are NOT filtered** — no `IsActive`/`IsDeleted` field is
  documented in RM's `model-Tenant.json` or any Owner model doc; confirmed with
  Johnny rather than inventing a filter criterion.
- **No Tenant+Unit join built.** The ticket's trap ("no combined embed exists,
  two calls + SA join") is a warning about RM's embed limitation, not a
  requirement for a merged tenant+unit view in this ticket's scope/acceptance
  criteria — tenants and units are returned as independent tagged lists.
- **Cache invalidation**: `?refresh=1` → `Cache::forget()` per connection key
  before recompute, same as the existing precedent. No version-counter (no
  write path in this ticket, unlike `TaskCacheVersion`).
- **`EnsureRmTokenValid` needs no change.** This closes the forward-looking
  flag from DEVSYM-404: the middleware only gates the calling superuser's own
  Sanctum+RM session, which is orthogonal to `PmConnectionContext` (used
  internally per-connection, each self-authenticating from stored credentials).
  DEVSYM-374 is the actual route wiring, and — moot given point above — no
  interaction/bug found.

## PR #141 review (3 findings, all addressed — one commit each)

- **Finding 1 (should-fix): silent partial data.** Counts silently under-reported
  when `CrossPmFanout` skipped a broken connection. Added
  `meta.connections.{reachable,reached,skipped}` (injected `SuperuserAccessPolicy`;
  `cachedResource` now returns `reached` = fan-out KEYS, so a reached-but-empty PM
  isn't miscounted as skipped). `reached` = succeeded for EVERY resource.
  Commit `a92ecc6`.
- **Finding 2 (nit): silent `MAX_PAGES` cap.** `fetchAllPages` now `Log::warning`s
  (resource + rm_location_id + total) when the cap — not the source total — ends
  the walk. Commit `e91be1c`.
- **Finding 3 (nit): duplicate embed.** `getActiveUnits` guarded the `Property`
  append with `str_contains`, matching the `ALWAYS_EMBED` pattern. Commit `2d9e5bb`.
- Recorded BEFORE addressing (PR-review protocol): correction file
  `2026-07-13_correction-pr-review-374-findings.md` + 3 RULES.md promotions
  (RM Integration = embed dedup; Code quality = no silent caps; Superuser stack
  ownership = surface reached/skipped on CrossPmFanout aggregations).

## Verification

- New/changed test files: 41 passed (144 assertions). After the review round:
  full suite 22 failed / 2 risky / 300 passed (+6 tests across the session),
  same `dev` baseline (22 failed / 2 risky), zero regressions.
- Full suite: 22 failed / 2 risky / 294 passed — confirmed identical baseline
  on `dev` (stashed and re-ran: 22 failed / 2 risky / 286 passed). Zero
  regressions; the pre-existing 22 failures are the known `DEV_AUTH_BYPASS`
  vs `assertUnauthorized()` issue documented in RULES.md.
- Pint clean, PHPStan level 5 clean on all touched files (run inside Docker).
- `php artisan l5-swagger:generate` run after adding the `OA\Get` attribute.

## Open questions

- None for 374. Flag for DEVSYM-376 (or whichever ticket resolves superuser
  auth/role): replace `PortfolioCacheService::CALLER_ROLE_PLACEHOLDER` with the
  authenticated caller's real `Role` once that's available.
