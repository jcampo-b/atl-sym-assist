## Review opened
- Timestamp: 2026-08-05
- Repo: BE (`SymAssist-Backend`)
- Branch: `fix/devsym-556-remove-local-discovery-shortcut`
- Base branch: `dev`
- Issue: DEVSYM-556

## Dev's stated focus
Extracted the RentManager location-discovery call out of `PmConnectionProvisioner::discoverLocations()`
into a new single-method interface (`RentManagerLocationDiscovery`) with a real Partner Token
implementation and a fake, bound in the existing module service provider — fake in `local`, real by
default. Removed an `APP_ENV=local` early-return that returned two hardcoded cpmomaha locations for
every corpid, along with its "TEMP — DO NOT COMMIT" comment. Added a `Log::error` to the previously
silent catch in `discoverLocations()` (no rethrow, still returns `[]`). Reworked the two provisioner
tests to inject an explicit test double instead of relying on the environment, and added one
characterization test for the throw path. Also corrected a section in
`docs/multi-tenant-default-variables-audit.md` that described the removed shortcut as safe.

## Files in scope
```
app/Modules/Integrations/IntegrationsServiceProvider.php
app/Modules/Integrations/RentManager/Services/Discovery/FakeLocationDiscovery.php
app/Modules/Integrations/RentManager/Services/Discovery/PartnerTokenLocationDiscovery.php
app/Modules/Integrations/RentManager/Services/Discovery/RentManagerLocationDiscovery.php
app/Modules/Integrations/RentManager/Services/PmConnectionProvisioner.php
docs/multi-tenant-default-variables-audit.md
tests/Feature/Modules/Integrations/RentManager/PmConnectionProvisionerTest.php
tests/Unit/Modules/Integrations/RentManager/Services/PmConnectionProvisionerRolesTest.php
```

## Verification run before findings
- `docker compose exec app php artisan test --filter=PmConnectionProvisioner` → **14 passed, 83 assertions**
- `./vendor/bin/pint --test` → **PASS (647 files)**
- `./vendor/bin/phpstan analyse --memory-limit=512M` → **[OK] No errors**
- `phpunit.xml:21` = `<env name="APP_ENV" value="testing"/>` with **no** `force="true"`;
  `docker-compose.yml:39,72` set `APP_ENV: local` → the doc's `APP_ENV` correction is accurate and
  the suite does run as `local`, so the container binds `FakeLocationDiscovery` during tests. Both
  rewritten tests override the binding with `$this->app->instance(...)`, so this is handled.
- Verified `app()->environment()` no longer appears anywhere in `PmConnectionProvisioner` — the
  ticket's core objective is met.
- Checked every other path that could reach discovery: only `RentManagerService::provisionPmConnections()`
  (→ `OnboardingInviteService:349`), and `PmInviteByInternalUserTest` mocks it at the
  `RentManagerService` level, so no other test silently inherits the fake.
- Verified in `vendor`: `Illuminate\Http\Client\RequestException::$truncateAt = 120` (default), which
  is why the `Str::limit(..., 500)` comment's `dontTruncate()` caveat is the accurate framing.

## Positive notes
The architecture is right: two real implementations behind the interface (not a speculative
abstraction — satisfies AGENTS.md's "confirm more than one concrete implementation today"), the
environment decision sits in the service provider, and the failure policy stayed in the caller so it
cannot diverge per implementation. That is exactly what RULES.md's "Environment is a detail of the
outermost ring" (DEVSYM-556's own entry) demands. `FakeLocationDiscovery` being corpid-keyed fixes
the real defect: an unknown corpid now returns empty instead of inheriting cpmomaha's locations.

## Findings

### should-fix

**File:** `app/Modules/Integrations/RentManager/Services/Discovery/PartnerTokenLocationDiscovery.php`

**Code snippet:**
```php
    public function forCorpid(string $corpid): array
    {
        $client = $this->client->withContext(new PartnerTokenCorpidContext($corpid));

        return (new RentManagerLocationService($client))->getCurrentLocations();
    }
```

**Severity:** `should-fix`

**What's wrong:** Line-for-line duplicate of `PmConnectionSeeder::discoverLocations()`
(`database/seeders/PmConnectionSeeder.php:113-124`), which still builds the client and calls RRM
directly instead of resolving the new interface.

**Why:** `code-review/SKILL.md` — "Before creating a new service, mapper, or helper — did you check
if an existing one already does this? If you copied logic from another file: stop. Extract it
instead, or use the original." Plus the rule this ticket itself added to RULES.md: *"Any local
capability that depends on a Partner Token call must be served by a fake bound in the container,
never by a real call."* The seeder is exactly such a capability and is reachable in `local` via
`PortfolioController`'s `seed` diag mode. The abstraction is half-applied: the provisioner is
fake-served in `local`, the seeder still 401s. `PmConnectionProvisioner`'s docblock still says
"Mirrors PmConnectionSeeder's discovery" — after this PR that mirroring IS the duplication.

**Suggested fix:** Seeder resolves `RentManagerLocationDiscovery`, keeping its own (legitimately
different) `$this->command?->warn(...)` failure policy. Reword/drop the "Mirrors PmConnectionSeeder's
discovery" docblock line. Fix touches a file outside this diff → follow-up ticket is a fair call.

**Comment for the dev:** The extraction is good, but `PmConnectionSeeder::discoverLocations()` (lines
113-124) is still the same call built by hand, so the seeder path keeps making a real Partner Token
call in `local` — the exact thing the new RULES.md entry from this ticket rules out. Could the seeder
resolve `RentManagerLocationDiscovery` and keep its own `command?->warn()` failure policy? Happy for
it to be a follow-up ticket if you'd rather keep this PR tight.

---

**File:** `app/Modules/Integrations/RentManager/Services/PmConnectionProvisioner.php`

**Code snippet:**
```php
            // Without this log a real RRM failure and "this corpid genuinely has
            // no granted locations" are indistinguishable — both reach the
            // caller as an empty array. Deliberately does NOT rethrow: failing
            // closed here changes behavior in the provisioning path and needs
            // characterization tests first (separate ticket, see DEVSYM-556).
```

**Severity:** `should-fix`

**What's wrong:** The comment points at DEVSYM-556 as the "separate ticket" — but this branch IS
DEVSYM-556, and it already adds the characterization test the comment calls a missing prerequisite.

**Why:** `AGENTS.md` code quality rules — comments carry a non-obvious WHY. This one is circular and
its precondition is stale: a maintainer follows the reference back to the ticket they are reading and
concludes the fail-closed change is still blocked on a test that
`test_logs_an_error_and_still_falls_back_to_pending_when_discovery_throws` now provides.

**Suggested fix:** Point at the real follow-up ticket (or say there isn't one yet) and drop the
"needs characterization tests first" clause. e.g. *"Deliberately does NOT rethrow — failing closed
changes provisioning behavior and is a separate decision; current behavior is pinned by
test_logs_an_error_and_still_falls_back_to_pending_when_discovery_throws."*

**Comment for the dev:** Small thing in the catch comment: it says the fail-closed change "needs
characterization tests first (separate ticket, see DEVSYM-556)" — but this is 556, and you added that
characterization test right here. Could you point it at the real follow-up ticket (or just say there
isn't one yet) and drop the prerequisite clause? Otherwise the next reader thinks the rethrow is
still blocked on work that's already done.

### nits

**File:** `app/Modules/Integrations/RentManager/Services/PmConnectionProvisioner.php`

**Code snippet:**
```php
            Log::error('pm_connection.location_discovery_failed', [
                'corpid' => $corpid,
                'exceptionClass' => $e::class,
                'exceptionMessage' => Str::limit($e->getMessage(), 500),
                'locationsReturned' => 0,
            ]);
```

**Severity:** `nit`

**What's wrong:** `'locationsReturned' => 0` is a hardcoded literal that can only ever be `0` on this
path; `500` is an unnamed bound.

**Why:** `code-review/SKILL.md` "constants that will never vary → inline them" / no dead code;
`AGENTS.md` "No magic constants with semantic meaning — name them." There is no success-side log to
contrast against, so `locationsReturned` adds nothing beyond the event name. On `500`: verified
`RequestException::$truncateAt = 120` in vendor, so for the RM failure this catch mostly sees the
limit never fires — the dev's own comment already says this, hence nit only.

**Suggested fix:** Drop `'locationsReturned' => 0`; if the bound stays, name it
(`private const MAX_LOGGED_MESSAGE_CHARS = 500;`).

**Comment for the dev:** Two tiny ones in the log context: `locationsReturned` can only ever be `0`
here so it isn't telling the reader anything, and `500` could be a named constant. The `Str::limit`
comment is genuinely useful though — I verified `RequestException::$truncateAt = 120`, so your note
about `dontTruncate()` being the case that matters is correct.

---

**File:** `tests/Feature/Modules/Integrations/RentManager/PmConnectionProvisionerTest.php`

**Code snippet:**
```php
     * The logging assertion is load-bearing: the log is the only thing that
     * tells a real RRM failure apart from "this corpid has no granted
     * locations", so deleting it must break a test rather than pass silently.
     */
```
```php
        Log::shouldHaveReceived('error')->once();
```

**Severity:** `nit`

**What's wrong:** The assertion guards against the log being *deleted* (what the comment literally
claims) but not against its payload being *degraded* — any `error` call with any message and any
context passes.

**Why:** `code-review/SKILL.md` tests section. The diagnostic value described in the comment lives in
the event name and the `corpid`/`exceptionClass` keys. Rename the event or drop `corpid` and this test
stays green while the thing it says it guards is gone.

**Suggested fix:** `Log::shouldHaveReceived('error')->once()->withArgs(fn ($event, $context) =>
$event === 'pm_connection.location_discovery_failed' && $context['corpid'] === 'brokencorp');`

**Comment for the dev:** The characterization test is a good addition and the `fakeDiscovery()` helper
asserting the corpid is the right call. One tightening on the log assertion:
`shouldHaveReceived('error')->once()` passes for *any* error log, so someone could rename the event or
drop `corpid` from the context and the test stays green. A `withArgs` on the event name + corpid would
make it match what the docblock says it's guarding.

## Considered and rejected (not filed)
- `FakeLocationDiscovery` living in `app/` rather than `tests/` — sanctioned by the RULES.md entry
  this ticket added ("served by a fake bound in the container").
- The interface itself as a speculative abstraction — two concrete implementations exist today, so it
  satisfies AGENTS.md's gate.
- `new RentManagerLocationService($client)` bypassing the provider's singleton — established
  precedent (the seeder does the same); the service is client-bound by construction.
- `bind` vs `singleton` for the discovery binding — `PmConnectionProvisioner` is resolved on demand
  and the dependency is cheap; `bind` is correct here.

## Verdict
**needs changes before merge** — no blockers and no correctness defect. The DEVSYM-556 self-reference
in the catch comment is a one-line fix; the seeder duplication is legitimately follow-up-ticket
material if the PR is to stay tight.

## SPACE log
- Time spent reviewing: 20 min (Johnny-provided)
- Model: Opus
- Findings: 0 blockers, 2 should-fix, 2 nits
- Verdict: needs changes before merge
