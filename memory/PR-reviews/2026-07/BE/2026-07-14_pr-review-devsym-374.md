## Review opened
- Timestamp: 2026-07-14
- Repo: BE
- Branch: feat/devsym-374-cross-pm-data-aggregation-and-caching-layer
- Base branch: dev
- Issue: DEVSYM-374

## Dev's stated focus
Cross-PM data aggregation and caching layer for the superuser Portfolio view.
Wraps DEVSYM-373's `CrossPmFanout`; caches each resource independently per
`rm_location_id` with SWR (`Cache::flexible`), tags every record with its
`rm_location_id`, and returns aggregate counts. New endpoint
`GET /api/superuser/portfolio` (middleware `api, auth:sanctum, rm_super_admin`).

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

## What was verified clean (no findings)
- Layer discipline: `PortfolioCacheService` (SA) calling RM mappers
  (`RentManagerPropertyMapper::toProperty`, etc.) is the DOMINANT codebase pattern
  (TaskService, PropertyService, OnboardingService, VacancyService, DailyPlannerService…).
  Not a violation. `new RentManager*Service($client)` is unique here but justified —
  the client is dynamic per-connection from the fan-out, can't be container-injected.
- RM traps handled: Status always embedded for issues (`ALWAYS_EMBED`), properties
  RM-filtered `IsActive,eq,true`, `Property.IsActive` filtered inside the RM layer
  (`getActiveUnits`), per-connection isolation via `CrossPmFanout` try/catch, no PATCH
  (Title trap N/A).
- Pagination `fetchAllPages`: `per_page` = requested `pageSize` (1000), not survivor
  count (confirmed in `normalizeListResponse`), so it walks source total correctly even
  when `getActiveUnits` filters rows out client-side. Terminates correctly; MAX_PAGES caps
  runaway loops.
- All service/mapper signatures verified: `getProperties`/`getTenants`/`getOwners`/
  `getServiceManagerIssues`, `RentManagerIssueService` 2-arg ctor, `CrossPmFanout::fanOut(Role, callable)`,
  `Role::PortfolioManager`. Test `match($endpoint)` paths align with real RESOURCE_PATHs
  (`properties`/`tenants` lowercase, `Units`/`Owners`/`ServiceManagerIssues` PascalCase — preexisting).
- Test reliability: `BypassAuthForDev` only fires on `environment('local')`; phpunit sets
  `APP_ENV=testing`, so the bypass never activates. Positive `actingAs(admin,'sanctum')`
  tests are correct; the negative test's `config(dev_auth_bypass=false)` is redundant-but-harmless.
- `CALLER_ROLE_PLACEHOLDER` has a thorough forward-reference comment to DEVSYM-415.
- Feature test asserts response shape; unit test covers tagging/counts/filters/caching/refresh.
- RESTClient `.http` includes 401/403/happy/refresh cases.

## Findings

### should-fix
**File:** `app/Modules/Superuser/Services/PortfolioCacheService.php` (counts block, ~L72–86)
```php
            'meta' => [
                'counts' => [
                    'properties' => count($properties),
                    'units' => count($units),
                    'tenants' => count($tenants),
                    'owners' => count($owners),
                    'tickets' => count($tickets),
                ],
            ],
```
When `CrossPmFanout` skips a broken connection (documented "may contain FEWER keys"),
aggregate counts silently under-report and the response has no partial-data signal.
A superuser can't distinguish "PM has 0 properties" from "PM #3 failed and was dropped".
Fan-out logs the skip server-side, but the API consumer gets nothing.
Fix: surface reached/skipped `rm_location_id`s in `meta` (e.g. `meta.connections.{reached,skipped}`),
or confirm it's a deliberate deferral.

### nit
**File:** `app/Modules/Superuser/Services/PortfolioCacheService.php` (L161–163)
```php
            $reachedEnd = $perPage <= 0 || ($page * $perPage) >= $total;
            $page++;
        } while (! $reachedEnd && $page <= self::MAX_PAGES);
```
When MAX_PAGES (50k rows) terminates the walk instead of reaching source total, partial
data is returned silently. Add a `Log::warning` (resource + `rm_location_id`) on that branch
so the "impossible" case is diagnosable.

### nit
**File:** `app/Modules/Integrations/RentManager/Services/RentManagerUnitService.php` (getActiveUnits, ~L96–97)
```php
        $existingEmbeds = isset($params['embeds']) ? $params['embeds'].',' : '';
        $params['embeds'] = $existingEmbeds.'Property';
```
No guard against a caller already embedding `Property` → `embeds=Property,Property`.
Diverges from the established `str_contains` embed-merge pattern (`RentManagerIssueMapper`
L125–127, `RentManagerUserMapper`). Inert for the current single caller, but the method is
public/reusable. Fix: append `Property` only when `! str_contains($existingEmbeds, 'Property')`.

## Verdict
ready to merge (no blockers; 1 should-fix worth a quick decision, 2 nits)

## SPACE log
- Time spent reviewing: TBD
- Model: Opus 4.8
- Findings: 0 blockers, 1 should-fix, 2 nits
- Verdict: ready to merge
