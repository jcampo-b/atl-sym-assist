# PR Review — DEVSYM-546 (BE)

## Review opened
- Timestamp: 2026-08-04
- Repo: BE (`SymAssist-Backend`)
- Branch: `feature/devsym-546-occupancy-collection-rate-kpi-fix`
- Base branch: `dev`
- Issue: devsym-546
- PR: https://github.com/Braintly/SymAssist-Backend/pull/315 — "fix(DEVSYM-546): derive occupancy total from a real unit count"
- Author: sebasaenz-braintly
- Commits: 1 (`717c8ca`)
- Reviewer model: Opus 5

## Dev's stated focus
From the PR description:

The Occupancy KPI's `total` was computed as merely `occupied + vacant` from two separate
Units queries filtered by `CurrentUnitStatus.UnitStatusType.IsVacant`. A unit whose status
isn't set in RM lands in NEITHER query, so it silently dropped out of the total — confirmed
live on cpmomaha/Ripper, where nearly every property reported `occupancy.total: 0` despite
genuinely having units.

This PR:
- Occupancy total now comes from `RentManagerService::countUnitsByProperty()`, the same
  real-per-property-unit-count call already trusted elsewhere over `Property.UnitCount`
  (documented as stale/unreliable on this RM account).
- Only ever raises total, never lowers it below what the vacancy-filtered passes already confirmed.
- Surfaces a warning when units with unknown status exist, or when the count itself hits RM's
  page-size cap.
- Corrects the class docblock, which claimed occupancy used
  `Property.OccupiedUnitCount/OccupancyUnitCount` — the code never actually did.

## Files in scope
```
app/Modules/Dashboard/Services/PropertyPerformanceService.php
tests/Unit/Modules/Dashboard/Services/PropertyPerformanceServiceOccupancyTest.php
```
Diffstat: 2 files changed, 168 insertions(+), 1 deletion(-)

## Verification run (reviewer, in Docker)
- `php artisan test --filter=PropertyPerformanceServiceOccupancyTest` → **3 passed, 11 assertions**
- `./vendor/bin/pint --test` → **PASS**, 643 files
- `./vendor/bin/phpstan analyse --memory-limit=1G` → 1 error, **pre-existing and unrelated**
  (`VoiceAIService::SYSTEM_PROMPT_V3 is unused`, file untouched by this PR)
- `php artisan test --filter=Dashboard` → 2 failed / 12 passed. Both failures
  (`AttentionNeededControllerTest::test_refresh_bypasses_the_cache`,
  `::test_returns_401_without_auth`) **reproduce on `origin/dev`** — verified by
  detached checkout. **No regression from this PR.**
- `phpstan.neon` `paths:` is `app/` only — `tests/` is NOT analysed. This is why the
  RULES.md "use `createMock`, not `Mockery::mock()`" rule was NOT raised as a finding:
  its stated motivation (PHPStan `argument.type` at level 5) cannot fire on a test file,
  and the sibling `tests/Unit/Modules/Properties/Services/PropertyServiceTest.php` mocks
  the same `RentManagerService` with Mockery.
- `RefreshDatabase` in a `tests/Unit` file was also NOT raised: 20+ of 73 files under
  `tests/Unit` already use it. Established convention, not a deviation.

## Findings

### Blockers

---

**File:** `app/Modules/Dashboard/Services/PropertyPerformanceService.php`
**Code snippet:**
```php
            $unknownStatusUnitsExist = false;
            foreach ($unitTotals['counts'] as $pid => $count) {
                $unitAggByProperty[$pid] ??= ['total' => 0, 'occupied' => 0, 'vacant' => 0];
                if ($count > $unitAggByProperty[$pid]['total']) {
                    $unknownStatusUnitsExist = true;
                    $unitAggByProperty[$pid]['total'] = $count;
                }
            }
```
…which feeds the unchanged rate formula in `computeOccupancy()`:
```php
        $rate = $total > 0 ? round($occupied / $total, 4) : null;
```
**Severity:** `blocker`
**What's wrong:** Raising `total` while leaving `occupied` at 0 turns `occupancy.rate` from
`null` ("unknown") into `0.0` ("0% occupied") for exactly the properties this PR is fixing.
**Why:** The PR states that on cpmomaha/Ripper "nearly every property reported
`occupancy.total: 0`" — meaning both vacancy passes returned nothing, so `occupied = 0` and
`vacant = 0`. After this change those properties get `total = N, occupied = 0` → `rate = 0.0`.
`computeSummary()` compounds it portfolio-wide
(`'occupancy_rate' => $units > 0 ? round($occupied / $units, 4) : null`): the headline
occupancy KPI goes from blank to ~0%. Trading a visibly-broken `0` total for a confidently
wrong `0%` rate is a worse failure mode — `null` tells the PM "we don't know", `0%` tells them
their building is empty. AGENTS.md ships this service with documented `meta.warnings` precisely
so unknowns are surfaced as unknown, not asserted.
**Suggested fix:** Decouple the denominators. Keep `rate` over *confirmed* units only —
`occupied / (occupied + vacant)`, `null` when that sum is 0 — and report the real count from
`countUnitsByProperty()` as `total`. Optionally expose the gap as its own field (e.g.
`unknown_status`) so the FE can render "N units unclassified" instead of implying they're
vacant. Same treatment for `computeSummary()`'s `occupancy_rate`.
**Comment for the dev:** (posted in review output — asks to split `rate` from `total` before merge)

### Should-fix

---

**File:** `app/Modules/Dashboard/Services/PropertyPerformanceService.php`
**Code snippet:**
```php
        if ($propertyIdSet !== []) {
            $unitTotals = $this->rentManager->countUnitsByProperty($propertyIdSet);
            if ($unitTotals['capped']) {
                $warnings[] = 'Unit count dataset hit the RM page-size cap — occupancy totals may be clipped.';
            }
```
…while Stage B, immediately above, already pools on the same dependency:
```php
        $stageB = [];
        if ($propertyIdSet !== []) {
            // Two filtered passes — RM's CurrentUnitStatus.UnitStatusType.IsVacant filter is
            // honored but the flag is not echoed, so we partition units by which pass produced them.
            $stageB['unitsOccupied'] = $propertyIdSet;
            $stageB['unitsVacant'] = $propertyIdSet;
        }
```
**Severity:** `should-fix`
**What's wrong:** The new RM call runs serially after Stage B instead of joining the Stage B
pool, even though it depends on nothing but `$propertyIdSet` — the exact dependency Stage B
is keyed on.
**Why:** This whole service is architected around two pooled stages (`poolDashboard`), with
in-code comments spelling out the intent ("Charges are fetched alongside charge-type discovery
… so they don't wait on it") and a `Benchmark`/`perf.dashboard.property_performance` profiling
hook around it. `RentManagerUnitService::unitsByCurrentVacancyRequest()` exists purely as a
pool descriptor, so the pattern for adding an intent is already established. Adding an
unpooled round-trip to the cold-compute path works against that design, and duplicates the
`$propertyIdSet !== []` guard three lines apart.
**Suggested fix:** Extract a `unitIdentitiesByPropertiesRequest()` descriptor in
`RentManagerUnitService` mirroring `unitsByCurrentVacancyRequest()`, register a
`unitIdentities` intent in `RentManagerService::poolDashboard()`, and add it to `$stageB`
alongside `unitsOccupied`/`unitsVacant`. Derive the counts and the `capped` flag from the
pooled response's `data` + `meta.pagination.total`. That folds the new work into the existing
guard block and drops the extra round-trip.

### Nits

---

**File:** `app/Modules/Dashboard/Services/PropertyPerformanceService.php`
**Code snippet:**
```php
            $unknownStatusUnitsExist = false;
            foreach ($unitTotals['counts'] as $pid => $count) {
                $unitAggByProperty[$pid] ??= ['total' => 0, 'occupied' => 0, 'vacant' => 0];
                if ($count > $unitAggByProperty[$pid]['total']) {
                    $unknownStatusUnitsExist = true;
                    $unitAggByProperty[$pid]['total'] = $count;
                }
            }
            if ($unknownStatusUnitsExist) {
                $warnings[] = 'Some units have no recorded occupancy status in Rent Manager — they count toward total but not toward occupied or vacant.';
            }
```
**Severity:** `nit`
**What's wrong:** Both the variable name and the warning assert a root cause the code can't
actually distinguish — all it observed is `count > occupied + vacant`.
**Why:** That delta has other causes. The vacancy passes are `emptyOn404 => true` in
`poolDashboard()`, so an RM 404 on either pass silently yields `[]` and trips this flag; the
same goes for the vacancy passes hitting their own `self::PAGE_SIZE` cap, which already emits
its own distinct warning eight lines up. A `meta.warnings` entry that names the wrong cause
sends a PM into RM to fix data that isn't broken.
**Suggested fix:** Rename to something the code can prove (e.g. `$totalRaisedAboveConfirmed`)
and soften the message to describe the observation rather than the diagnosis. Keep the causal
explanation in the code comment, which is the right place for it.

---

**File:** `app/Modules/Dashboard/Services/PropertyPerformanceService.php`
**Code snippet:**
```php
                'maintenance' => $this->computeMaintenance(
                    $issuesByProperty[$pid] ?? [],
                    $maintCostRows,
                    $maintenanceSource === 'bill_detail' ? 'amount' : 'cost',
                    $occAgg['total'],
                ),
```
**Severity:** `nit`
**What's wrong:** `maintenance.cost_per_unit` (and `computeSummary`'s `maintenance_avg_cost`)
share the `total` denominator, so their values shift too — not mentioned in the PR description.
**Why:** RULES.md / code-review skill — "If this PR changes an API contract — is the breaking
change explicitly flagged in the PR description?" No shape change here, but three KPI values
move, and `cost_per_unit` goes from `null` to a real number for every property that previously
had `total = 0`. FE/QA should know which numbers to re-baseline.
**Suggested fix:** One line in the PR summary. No code change.

---

**File:** `tests/Unit/Modules/Dashboard/Services/PropertyPerformanceServiceOccupancyTest.php`
**Code snippet:**
```php
     */
    private function service(array $poolResponses, array $unitTotals): PropertyPerformanceService
    {
```
**Severity:** `nit`
**What's wrong:** The docblock documents `$poolResponses` but not `$unitTotals`.
**Why:** Consistency — the helper's other param is documented, and the shape here
(`array{counts: array<int, int>, capped: bool}`) is exactly the contract the test is pinning.
**Suggested fix:** Add `@param array{counts: array<int, int>, capped: bool} $unitTotals`.

## What the review confirmed as correct (not findings)
- The root-cause diagnosis is right and evidenced live (cpmomaha/Ripper).
- "Only ever raises total" is genuinely implemented (`if ($count > ...['total'])`) and is the
  correct mitigation for `countUnitsByProperty()`'s own batch-wide page-size cap
  (documented in `RentManagerService::countUnitsByProperty()`: a property's count dropped
  from 55 to 1 once bundled into a 1000-property batch).
- Docblock correction is accurate — the code never read
  `Property.OccupiedUnitCount/OccupancyUnitCount`.
- No layer violation: no PascalCase RM field reaches the SA layer; the count is obtained
  through `RentManagerService`, and `PropertyService` already establishes that precedent.
- Deleted-unit trap (AGENTS.md) does NOT apply here: `propertiesRequest(null)` filters
  `IsActive,eq,true`, so `$propertyIdSet` is already active-scoped, and both the vacancy
  passes and the identities call use the same `PropertyID,in,(...)` scope. Not introduced
  by this diff.
- `$counts = array_fill_keys($propertyIds, 0)` means a property with 0 counted units cannot
  lower an existing agg — verified safe.

## Verdict
**needs changes before merge** — the `total` fix is correct and well-evidenced; the `rate`
fallout (blocker) needs a product/design decision first.

## SPACE log
- Time spent reviewing: **90 min** (Johnny — reading + thinking, TOTAL across all 3 passes;
  supersedes the 30 min recorded after pass 1)
- Model: Opus 5
- Passes: 3 (initial review → `fe0c9fd` pooling refactor → `4dc21a7` blocker fix)
- Findings: 1 blocker, 1 should-fix, 4 nits
- Resolved by the dev: the blocker, the should-fix, 1 nit
- Waived by Johnny: the 3 remaining nits (PR description, `$unknownStatusUnitsExist` naming,
  duplicate cap-check techniques) — explicitly not worth the round trip
- Verdict: ready to merge; FE needs a heads-up about `occupancy.unknown_status` and
  `meta.totals.unknown_status_units`

---

## Review closed
- Timestamp: 2026-08-04 16:43 -03
- Resolution: **approved** (verified via `gh pr view 315`: `state=OPEN`,
  `reviewDecision=APPROVED`, `mergedAt=null` — approved, merge pending at close time)
- Time spent reviewing: 90 min total (Johnny). His time reading/validating the review itself is
  folded into that figure, not tracked separately — he confirmed 90 as the whole number.

### Findings addressed
- **Blocker — occupancy rate flipping `null` → `0.0`: FIXED** in `4dc21a7`. `computeOccupancy()`
  and `computeSummary()` both divide by confirmed units (`occupied + vacant`), `null` when zero.
  New `occupancy.unknown_status` / `meta.totals.unknown_status_units` surface the gap.
  Net effect: `rate` is bit-identical to `dev`, so the PR ended up purely corrective on `total`
  and purely additive on the response shape.
- **Should-fix — unpooled serial RM call: FIXED** in `fe0c9fd`. Extracted
  `unitIdentitiesByPropertiesRequest()`, registered a `unitIdentities` intent in
  `poolDashboard()`, folded into Stage B. Cap detection verified equivalent post-refactor.
- **Nit — missing test `@param`: FIXED** in `fe0c9fd` (parameter removed entirely).

### Findings waived
Johnny explicitly waived the remaining three as not worth the round trip:
1. PR description left stale (still cites `countUnitsByProperty()`; omits the confirmed-only rate
   and the two new fields; test plan says 3 tests, there are 5).
2. `$unknownStatusUnitsExist` naming + warning assert a cause the code can't distinguish from an
   `emptyOn404` vacancy pass or a clipped one.
3. Two cap-check techniques five lines apart — the new `meta.pagination.total` comparison is
   exact; the older vacancy check infers a cap from `count(rows) >= self::PAGE_SIZE`, which only
   holds because SA's `PAGE_SIZE` coincidentally equals `OCCUPANCY_PAGE_SIZE`.

### Follow-ups leaving this review
- **FE heads-up owed** (replaces waived nit #1's practical purpose): `/api/dashboard/property-performance`
  now returns `occupancy.unknown_status` per property and `meta.totals.unknown_status_units` on
  `/summary`. Additive, non-breaking — but if FE never renders it, the unclassified-units gap this
  PR set out to expose stays invisible on the dashboard.
- **Spun off as its own task** (out of scope — file not in the Step 1b list):
  `PropertyPerformanceController.php`'s OpenApi `index` description still reads
  "Occupancy rate (pre-computed by RM)", the same false claim `717c8ca` corrected in the class
  docblock. Its `responses:` carry no schema, so the two new fields cause no schema drift.
- If cap-check nit #3 ever bites, the fix is to extend the new `meta.pagination.total`-vs-rows
  comparison to the two vacancy passes and collapse the two warnings into one.

## Status
**CLOSED** — approved 2026-08-04. See "Review closed" at the end of this file.

---

## Re-review pass 2 — commit `fe0c9fd` "refactor: pool the real unit count instead of a serial call"

Scope grew to 4 files (added `RentManagerService.php`, `RentManagerUnitService.php`).

**Resolved:** the pooling should-fix. Extracted `unitIdentitiesByPropertiesRequest()` in the RM
layer, had `getUnitIdentitiesByProperties()` consume it (no duplicated query builder), registered
a `unitIdentities` intent in `poolDashboard()` with the same `emptyOn404` + `toUnitIdentity`
mapper as its siblings, and reused the existing `groupBy()` helper instead of a new counter.
Serial round-trip gone; duplicate `$propertyIdSet !== []` guard collapsed into Stage B.

**Verified the refactor did not break cap detection** (the real risk in this move):
`normalizePooledResponse()` reads `x-total-results` and passes it to `normalizeListResponse()`
as `$externalTotal`, landing at `meta.pagination.total` — identical to the path the old
`countUnitsByProperty()` used for `$capped = $total > count($items)`. The `?? count(...)`
fallback is also correct: `normalizeListResponse` omits `meta` entirely when total and rows are
both 0, and `0 > 0` is false, so no spurious warning.

**Resolved:** the test-docblock `@param` nit, by dropping the second parameter entirely.

**Still open after this pass:** the blocker (untouched), the `$unknownStatusUnitsExist` nit,
the PR-description nit.

**New nit raised:** two cap-check techniques five lines apart — the new one compares
`meta.pagination.total` against rows returned (exact); the older vacancy check infers a cap from
`count(rows) >= self::PAGE_SIZE`, which only works because SA's `PAGE_SIZE = 1000` happens to
equal `RentManagerUnitService::OCCUPANCY_PAGE_SIZE = 1000` — two unrelated constants nothing
keeps in sync.

**Process note (cost Johnny a round trip):** Johnny read the `717c8ca...fe0c9fd` compare view,
saw `$unitAggByProperty[$pid]['total'] = $count;` → `= count($rows);` on the blocker's anchor
line, and concluded the blocker was fixed. It wasn't — the line changed for the pooling refactor,
same assignment, different source. **A line appearing in the diff is not evidence the finding was
addressed.** Fastest check: find the test assertion that pins the behaviour you objected to. If
that assertion didn't change, the behaviour didn't change. Here
`PropertyPerformanceServiceOccupancyTest:69` still read `assertSame(0.0, $occupancy['rate'])`.
Corollary: when a later commit edits the line a review comment is anchored to, GitHub marks the
comment **outdated** and collapses it — the dev may never see it. Re-anchor on the new line.

---

## Re-review pass 3 — commit `4dc21a7` "fix: occupancy rate is confirmed-units-only, not occupied/total"

### Blocker: RESOLVED

`computeOccupancy()` now computes `$confirmed = $occupied + $vacant` and divides by that,
`null` when it's 0. `computeSummary()` accumulates `$vacant` and applies the same
confirmed-only denominator. New `occupancy.unknown_status` and
`meta.totals.unknown_status_units` surface the gap. `maintenance_avg_cost` and
`computeMaintenance()`'s `cost_per_unit` correctly left dividing by the real unit count.
Docblocks updated in three places (class-level v1 limits, `computeSummary` KPI list,
`computeOccupancy`).

**Key verification — the fix leaves `rate` bit-identical to `dev`.** On `dev`,
`$unitAggByProperty[$pid]['total']` is written ONLY by `aggregateUnitsByProperty()`
(`git grep -c "unitAggByProperty[\$pid]['total'] =" origin/dev` → 0 matches; the only writer is
`$out[$pid]['total']++` at line 541), so dev's `total` IS `occupied + vacant`, and dev's
`round($occupied / $total, 4)` IS the confirmed-only rate. Both null-branches match too
(dev `total > 0`, new `confirmed > 0`). So the final state of this PR is **purely corrective on
`total` and purely additive on the response shape — zero change to `rate` versus `dev`.** No
product decision is needed anymore; the KPI-regression risk is gone.

Tests: the pinned assertion flipped to `assertNull($occupancy['rate'])` plus
`assertSame(1, $occupancy['unknown_status'])`, and two genuinely new cases were added —
`test_rate_divides_by_confirmed_units_only_not_total` (mixed confirmed/unconfirmed at one
property: 1 occupied / 2 confirmed = 0.5 with total 3) and
`test_summary_occupancy_rate_divides_by_confirmed_units_only_across_the_portfolio` (the
compounding path). 5 tests / 22 assertions.

### Considered and deliberately NOT raised
- `max($total - $confirmed, 0)` / `max($units - $confirmed, 0)` are unreachable given the
  invariant (`aggregateUnitsByProperty` seeds `total` as `occupied + vacant`; the merge only
  raises it). Not flagged: `computeOccupancy()` already clamps all three inputs with
  `max(..., 0)`, so this matches the function's existing defensive posture. Flagging it would be
  pedantic and inconsistent with the file.
- RULES.md "every modified endpoint needs a feature test asserting response shape" is satisfied
  here. `PropertyPerformanceInternalUserTest` mocks `PropertyPerformanceService` outright — it
  asserts routing/auth/PM-connection scoping, so it *cannot* assert the KPI shape; the mock
  supplies the payload. The shape lives in the service and is asserted by the unit tests, which
  now cover both new fields.

### Still open (none blocking)
1. **PR description — the one I'd insist on.** Now stale on three counts: it still says
   "total now comes from `RentManagerService::countUnitsByProperty()`" (it's the pooled
   `unitIdentities` intent since `fe0c9fd`); it never mentions the confirmed-only rate or the two
   new response fields, which is the part FE actually needs; and the test plan lists 3 unit tests
   when there are now 5.
2. `$unknownStatusUnitsExist` naming/warning asserts a cause the code can't distinguish.
3. Two cap-check techniques five lines apart.

### Out of scope — file NOT in the Step 1b list, spun off instead of filed as a finding
`PropertyPerformanceController.php`'s OpenApi `index` description still reads
"Occupancy rate (pre-computed by RM)" — the exact false claim commit `717c8ca` set out to fix in
the class docblock, living on in the Swagger description FE devs actually read. The OpenApi
`responses` carry no schema (descriptions only), so the two new fields cause no schema drift.

## Verdict (pass 3)
**ready to merge once the PR description is updated.** Code is clean: 5/5 occupancy tests,
33 passed across the affected suites, Pint PASS (647 files), PHPStan no new errors (the single
`VoiceAIService::SYSTEM_PROMPT_V3 is unused` is pre-existing on `dev`), and the 2
`AttentionNeededControllerTest` failures reproduce on `dev`.
