## Review opened
- Timestamp: 2026-07-13 21:24 -03
- Repo: BE
- Branch: feat/devsym-374-cross-pm-data-aggregation-and-caching-layer
- Base branch: dev (4 local commits, not pushed; working tree clean)
- Issue: DEVSYM-374

## Dev's stated focus
Cross-PM cache + aggregation layer for the superuser Portfolio view: HTTP endpoint
(`GET /api/superuser/portfolio`), `PortfolioCacheService`, `locationId()` getter on
`RentManagerHttpClient`, `getActiveUnits()` on the RM Unit service, and tests.

## Files in scope
```
RESTClient/superuser-portfolio.http
app/Modules/Integrations/RentManager/Services/RentManagerHttpClient.php
app/Modules/Integrations/RentManager/Services/RentManagerUnitService.php
app/Modules/Superuser/Controllers/PortfolioController.php
app/Modules/Superuser/Requests/PortfolioRequest.php
app/Modules/Superuser/Services/PortfolioCacheService.php
app/Modules/Superuser/SuperuserServiceProvider.php
app/Modules/Superuser/routes.php
bootstrap/providers.php
storage/api-docs/api-docs.json
tests/Feature/Modules/Superuser/PortfolioControllerTest.php
tests/Unit/Modules/Integrations/RentManager/Services/RentManagerHttpClientTest.php
tests/Unit/Modules/Integrations/RentManager/Services/RentManagerUnitServiceTest.php
tests/Unit/Modules/Superuser/Services/PortfolioCacheServiceTest.php
```

## Findings

### should-fix

**File:** `app/Modules/Superuser/Services/PortfolioCacheService.php`
Snippet:
```php
    private const PAGE_SIZE = 1000;
```
```php
        $result = (new RentManagerPropertyService($client))->getProperties([
            'filters' => 'IsActive,eq,true',
            'pageSize' => self::PAGE_SIZE,
        ]);
```
What's wrong: single page capped at 1000 rows per resource per connection, no pagination
loop, `meta.pagination.total` ignored → silent truncation for any PM over 1000 records.
Why: this is the superuser cross-PM source-of-truth view; silent data loss in an
aggregation endpoint is a correctness gap (tenants especially can exceed 1000). The RM
list responses already expose the total, so the ceiling is known but unused.
Suggested fix: loop pages until total is reached, or — if a hard cap is deliberate —
surface `truncated`/`total` per resource in `meta` so it is not silent.

### nits

**File:** `app/Modules/Superuser/Services/PortfolioCacheService.php`
Snippet:
```php
            'meta' => [
                'counts' => [
                    'properties' => count($properties),
                    'units' => count($units),
                    'tenants' => count($tenants),
                ],
            ],
```
What's wrong: response ships 5 lists (properties/units/tenants/owners/tickets) but
`meta.counts` reports only 3 — owners and tickets have no count.
Why: asymmetric contract; open-tickets count is the most operationally useful number for
a portfolio view. Likely oversight.
Suggested fix: add owners + tickets to counts, or confirm intentional.

**File:** `tests/Unit/Modules/Superuser/Services/PortfolioCacheServiceTest.php`
Snippet:
```php
        $client->method('getResponse')->willReturnCallback(fn (string $endpoint) => match ($endpoint) {
            'properties' => $this->jsonResponse([
                ['PropertyID' => 1, 'Name' => 'Sunset Apartments', 'IsActive' => true],
            ]),
```
What's wrong: mock callback matches on `$endpoint` only, ignores `$params`; no test
asserts `fetchProperties` sends `filters => IsActive,eq,true` or `fetchTickets` sends
`IsClosed,eq,false` + `Status` embed.
Why: dropping the active/open filter would regress silently with all tests green.
Suggested fix: capture and assert the `$params` argument for at least one resource.

## Notes / verified-not-a-finding
- Mapping RM data in the SA layer (`RentManagerPropertyMapper::toProperty(...)` etc.) is
  the ESTABLISHED pattern — PropertyService/TaskService/UnitService/OnboardingService all
  do it. Not a violation.
- Passing an RM filter string (`IsActive,eq,true`) from an SA service is also established
  (`OnboardingService` does it). Not a leak.
- Test endpoint casing (`properties`/`tenants` lowercase vs `Units`/`Owners`/`ServiceManagerIssues`
  PascalCase) matches the real service `getResponse` targets/RESOURCE_PATH constants. Correct.
- 403 gate test is valid: `BypassAuthForDev` only fires on `environment('local')` +
  config flag + no bearer token; the 403 comes from `EnsureIsSuperAdmin` comparing
  `rm_user_id`. RULES.md rule 138 (`withoutMiddleware`) does not strictly apply — these
  tests mock the service and don't read authed-user properties.
- `getActiveUnits()` correctly implements the RULES.md embed-filter-leak correction.

## Verdict
needs changes before merge (1 should-fix; 2 nits)

## Re-review (fixes verification) — 2026-07-13 21:xx -03
Dev squashed fixes onto the original commits (nothing pushed; only final state reviewable).
Targeted verification of the three prior findings + scope-creep check.

1. Pagination (should-fix) — RESOLVED. New `fetchAllPages()` walks `pageNumber` 1..N,
   accumulates `data`, terminates on `($page * per_page) >= total` with `MAX_PAGES = 50`
   safety cap against a missing/garbage `total`. Confirmed `per_page` in the normalized
   meta = the REQUESTED `pageSize` (`PerformsRentManagerRequests::normalizeListResponse`
   line 167: `'per_page' => $pageSize`), and `total` = source `x-total-results` header
   (untouched by `getActiveUnits`' client-side row filtering). So it counts requested
   pages against the source total — correct even when the RM layer filters rows out.
   Termination reads only snake_case `meta.pagination.*`; no PascalCase RM field touched
   in the SA layer. RM-layer diff unchanged (only `locationId()` + `getActiveUnits()`),
   `Property.IsActive` filtering still inside Integrations.

2. meta.counts (nit) — RESOLVED. Now reports all 5 resources (properties, units, tenants,
   owners, tickets). `test_compute_..._reports_counts` asserts all 5.

3. Filter assertion (nit) — RESOLVED. New `test_compute_sends_the_active_filters_to_rm`
   captures the `$params` arg per endpoint and asserts `filters === 'IsActive,eq,true'`
   (properties) and `'IsClosed,eq,false'` (ServiceManagerIssues) — not just the endpoint.

4. Scope-creep check — CLEAN except one stale doc:
   - `RESTClient/superuser-portfolio.http` line 37 still says
     `meta.counts.{properties,units,tenants}` — not updated to the now-5 counts. New nit
     (doc drift). Snippet:
     ```
     # rm_location_id; meta.counts.{properties,units,tenants}.
     ```
   - `storage/api-docs/api-docs.json` diff is only the portfolio operation (OA regen).
   - No new files, no unrelated changes.

Re-review verdict: all 3 findings resolved; 1 new trivial nit (stale .http comment).
Ready to merge once the .http comment is updated (or waved through).

## SPACE log
- Time spent reviewing: TBD
- Model: Opus 4.8
- Findings (round 1): 0 blockers, 1 should-fix, 2 nits — all resolved
- Findings (round 2 / re-review): 1 nit (stale .http counts comment)
- Verdict: ready to merge (pending trivial .http doc fix)
