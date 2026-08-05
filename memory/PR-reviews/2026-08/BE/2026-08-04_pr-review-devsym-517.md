# PR Review — DEVSYM-517 (BE) — round 2

## Review opened
- Timestamp: 2026-08-04
- Repo: BE (`SymAssist-Backend`)
- Branch: `fix/devsym-517-superuser-daily-planner-ptr-tasks` (`b494c5f`)
- Base branch: `dev`
- Issue: DEVSYM-517 — [BE] Superuser: Daily Planner no muestra tareas de symassist-PTR
- Prior review: `.atl/memory/PR-reviews/2026-07/BE/2026-07-31_pr-review-devsym-517.md` (PR #313, commit `7454a13`)

## Dev's stated focus
Not supplied in the review request. Derived from verifiable sources rather than assumed:
the round-1 focus statement in the prior review file, plus the new commit
`b494c5f fix(DEVSYM-517): merge the superuser daily planner across every corpid` — which is
exactly what round 1 listed as *deliberately out of scope*.

Round 1 focus (still applicable): a superuser's Daily Planner was always empty because
`TaskService::pending()` resolved the assignee from `Auth::user()->rm_user_id`, which an
`InternalUser` has no column for. It now resolves to the `symassist-PTR` RM user id stored on
`pm_connections.sa_ptr_rm_user_id`.

Round 2 delta: `pending()` no longer resolves a single `ActingPmConnection`'s PTR id — it
delegates to a new private `pendingForSuperuser()` that fans out across EVERY active
connection with a provisioned PTR id, tags each task with `pm_connection_id`, and merges.

Branch commits:
```
b494c5f fix(DEVSYM-517): merge the superuser daily planner across every corpid
9a23cf7 test(DEVSYM-517): pin the superuser daily planner assignee resolution
81219a9 fix(DEVSYM-517): resolve the daily planner assignee from the acting connection's PTR id
dddd776 fix(DEVSYM-517): persist the symassist-PTR RM user id per PM connection
```

## Files in scope
```
app/Models/PmConnection.php
app/Modules/Tasks/Services/TaskService.php
database/migrations/2026_07_31_000001_add_sa_ptr_rm_user_id_to_pm_connections_table.php
tests/Feature/Modules/InternalUsers/InternalUserPmContextEndpointsTest.php
```
4 files, 372 insertions, 3 deletions.

## Round-1 findings — status
- **should-fix (silent empty planner, no log)** → addressed. `Log::warning('task_service.pending.no_ptr_connections_provisioned')` added, with a test asserting it fires and a second test asserting it does NOT fire for a PM with a null `rm_user_id`. See round-2 should-fix #3 for the remaining partial-provisioning gap.
- **nit (migration docblock said "per-corpid" for a per-connection column)** → addressed. The docblock now carries an `IMPORTANT` block spelling out the per-row replication, matching the rule promoted to `RULES.md`.
- **nit (`nullsafe.neverNull` comment looked unfounded)** → addressed. Empirically re-verified; the real mechanism is now documented in `RULES.md` (the rule fires on the syntactic position, not on receiver nullability). The comment in question no longer exists in the diff — the whole expression was replaced by the `pendingForSuperuser()` delegation.

## Findings

### Blockers

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
        $cacheKey = 'tasks_list_ptr_v'.TaskCacheVersion::current().'_'.md5((string) json_encode([
            $connections->pluck('id')->sort()->values()->all(),
            $data->page, $data->perPage, $data->pageNumber, $data->pageSize,
        ]));
```
```php
                    $result = $this->rejectScheduledTasks($this->fetchList($rmParams));
```
And the contract the filtered method itself declares:
```php
     * Applied outside the list cache so scheduling/unscheduling a task is
     * reflected immediately. Meta is preserved from the underlying list call.
```

**Severity:** `blocker`

**What's wrong:** A per-user filter (`rejectScheduledTasks()`) runs *inside* a cache entry whose
key carries no actor, so one superuser's already-scheduled tasks are removed from every other
superuser's planner for up to 5 minutes.

**Why:**
- `rejectScheduledTasks()` → `DailyPlannerScheduleService::scheduledTaskIdsForUser()` →
  `preferenceOwner()`, which for an `InternalUser` returns
  `DailyPlannerActingPreference::firstOrCreate(['internal_user_id' => $user->id, 'pm_connection_id' => $connection->id])`.
  The result is per internal user.
- Its own docblock (quoted above) states the filter is applied *outside* the list cache
  precisely so scheduling is reflected immediately. `pending()`'s PM branch honours that:
  `sortByDueDateAscending($this->rejectScheduledTasks($result))` after `cachedFetchList()`.
  This branch does the opposite.
- Every other domain list key in the repo carries the actor: `cachedFetchList()` uses
  `'tasks_list_'.Auth::id().'_c'.$this->connectionCacheKeySuffix().'_v'...`, same in
  `listPropertyScopedOrAssignedToMe()`, `TaskStatsService`, `DailyPlannerService`.
  `RULES.md` → Caching layer as-built: *"All domain keys are `Auth::id()`-based."*
- `TaskCacheVersion::bump()` is never called from `DailyPlannerScheduleService`, so scheduling
  a slot cannot invalidate this entry either.

**Failure scenario:** Superusers A and B both have the PTR bucket. A schedules task 700 into
their planner → A's pending list correctly drops it, and the filtered array is cached under
`tasks_list_ptr_v3_<hash>`. Within the 5-minute TTL, B calls `GET /api/tasks/pending` → cache
hit → task 700 is absent from B's planner too, even though B never scheduled it. Symmetrically,
B schedules 701 and keeps seeing it in pending for up to 5 minutes because the cached array was
built before B's slot existed.

**Suggested fix:** Cache only the raw merged fan-out (the `fetchList()` rows tagged with
`pm_connection_id`) and apply `rejectScheduledTasks()` after the cache, mirroring `pending()`'s
PM branch. Note the status whitelist inside `rejectScheduledTasks()` is per-corpid
(`rmStatusIdsExcludingCompleted()` resolves against the acting connection), so the post-cache
pass has to re-point `ActingPmConnection` per distinct `pm_connection_id` group rather than run
once flat. Adding the internal user id to the key is worth doing as defence in depth, but on its
own it fixes the bleed and leaves the documented staleness contract broken.

**Comment for the dev:** The merge looks right, but `rejectScheduledTasks()` ended up inside the
`Cache::remember()` closure, and that key has no actor in it — unlike `cachedFetchList()` /
`listPropertyScopedOrAssignedToMe()`, which both carry `Auth::id()`. Since `preferenceOwner()`
resolves per `internal_user_id`, the cached array is one superuser's filtered view served to all
of them for the 5-minute TTL, and nothing bumps `TaskCacheVersion` when a slot is scheduled.
`rejectScheduledTasks()`'s own docblock says it's applied outside the list cache so scheduling
shows up immediately — could we cache just the raw merged rows and filter after? Heads-up that
the status whitelist inside it is per-corpid, so the post-cache pass needs to walk the
connections again rather than run flat once.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
                foreach ($connections as $connection) {
                    $actingConnection->set($connection);

                    // Status catalog ids are per-corpid (RM assigns them
                    // independently per tenant — the exact DEVSYM-461 trap
                    // this ticket's own "known traps" section warns about), so
                    // this must resolve AFTER pointing ActingPmConnection at
                    // this connection, never once before the loop.
                    $pendingData = new TaskListData(
```

**Severity:** `blocker`

**What's wrong:** The cross-connection fan-out has no per-iteration `try/catch`, so one broken PM
connection aborts the whole superuser planner.

**Why:** `RULES.md` → RM Integration, verbatim: *"Any fan-out / loop over N independent RM targets
(connections, tenants, vendors) must isolate each iteration in `try/catch (\Throwable)`, log
sanitized (`exceptionClass` = `$e::class` + `exceptionMessage` only — NEVER the full Throwable, RM
bodies carry PII), skip the failed target, and accumulate partial results. One broken target must
not abort the rest."* That rule's own stated origin is this exact failure class —
`PmConnectionContext::decodeCredentials()` throwing `RentManagerAuthenticationException` on a
null/malformed `rm_credential_ref`. The repo already has the compliant implementation to mirror:
`CrossPmFanout::fanOut()` (`app/Modules/Superuser/Services/CrossPmFanout.php:54-67`), which is the
established superuser fan-out and does exactly this.

**Failure scenario:** Three corpids provisioned with a PTR id. The second one's Partner Token grant
is revoked in PWA (or its `rm_credential_ref` is malformed) → the throw escapes the `foreach`,
escapes `Cache::remember`, and `GET /api/tasks/pending` returns a 500. The superuser's Daily Planner
is empty for *all three* corpids — the exact symptom DEVSYM-517 was filed for, now triggerable by
any single unhealthy PM.

**Suggested fix:** Wrap the loop body in `try { ... } catch (\Throwable $e) { Log::warning(...); }`
with `pmConnectionId` + `exceptionClass` + `exceptionMessage` only, and continue to the next
connection. Copy the shape from `CrossPmFanout::fanOut()`. Add a test where one connection's RM call
throws and the others still return.

**Comment for the dev:** This fan-out needs the per-item isolation rule from `RULES.md` —
`try/catch (\Throwable)` per connection, sanitized log (class + message only, never the full
Throwable), skip and keep going. `CrossPmFanout::fanOut()` already does it and its docblock explains
why. Right now one PM with a revoked Partner Token grant or a bad `rm_credential_ref` 500s the whole
endpoint, which leaves the superuser with an empty planner across every corpid — the same symptom
this ticket fixes. A test with one throwing connection would pin it.

---

### Should-fix

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
        return $this->sortByDueDateAscending([
            'data' => $perPage !== null ? array_slice($tasks, 0, $perPage) : $tasks,
            'meta' => [
                'pagination' => [
                    'current_page' => $page ?? RentManagerIssueMapper::FIRST_PAGE_NUMBER,
                    'per_page' => $perPage ?? count($tasks),
                    'total' => count($tasks),
                ],
            ],
        ]);
```

**Severity:** `should-fix`

**What's wrong:** The merged list is truncated to `per_page` in fan-out order and sorted only
afterwards, so once the first connection fills the page no later corpid's task can ever surface.

**Why:** `sortByDueDateAscending()` receives the already-sliced array, so the sort reorders a page
that was selected by DB iteration order rather than by due date. `meta.total` still reports the full
merged count, so the response advertises rows that are unreachable at any page. This also
contradicts this method's own docblock — *"a superuser's Daily Planner is the symassist-PTR bucket
merged across EVERY corpid"*. `listPropertyScopedOrAssignedToMe()`, which the pagination note cites
as precedent, applies no cross-source sort at all, so it does not license slice-before-sort here.

**Failure scenario:** `per_page=10`. Corpid A has 12 PTR tasks all due 2026-09-30; corpid B has 1 due
today. `$merged` = 13 rows, A's first. `array_slice(..., 0, 10)` keeps only A's rows, then the sort
orders those 10 by due date. B's task due *today* never renders, while `total: 13` tells the FE there
is more.

**Suggested fix:** Sort the merged array before slicing — run `sortByDueDateAscending()` on the full
`$tasks`, then `array_slice`. Extend `test_pending_merges_ptr_tasks_across_every_connection_with_a_ptr_user_id`
with a `per_page` smaller than the first connection's row count.

**Comment for the dev:** Small ordering issue with real teeth: `array_slice()` runs before
`sortByDueDateAscending()`, so the page is picked in fan-out order and only then sorted. With
`per_page=10` and 12 tasks on the first corpid, a task due today on the second corpid can never
appear while `total` still says 13. Sorting the full merged array first and slicing after fixes it —
worth a test case with `per_page` below the first connection's count.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
        $connections = PmConnection::query()->active()->whereNotNull('sa_ptr_rm_user_id')->get();
```
Alongside the dedup guidance in this method's own docblock:
```php
     * is only unique within a corpid, so each task is tagged with
     * pm_connection_id: two different corpids can legitimately return the
     * same bare id, and merging (or any future dedup) must key on the pair,
     * never the id alone.
```

**Severity:** `should-fix`

**What's wrong:** The fan-out iterates connection *rows*, but the PTR id is a corpid-level value
replicated across every row of that corpid — so a multi-location corpid is queried once per location
with an identical assignee filter, and nothing dedupes the result.

**Why:** `RULES.md` → DEVSYM-399 as-built, verbatim: *"`pm_connections` holds one row per (corpid,
location) ... any value that is semantically per-corpid but stored on this table (`sa_*_role_id`,
`sa_ptr_rm_user_id`) must be written to EVERY row of that corpid."* The migration in this same PR
mandates it: *"the SAME id must be written to EVERY row of that corpid."* And
`PmConnection::forCorpidAndLocation()`'s docblock (DEVSYM-503) confirms the multi-location case is
real, not theoretical: *"a user with valid access to more than one location of the same corpid
authenticates successfully against ALL of them."* The docblock's own dedup rule does not cover this:
same corpid with two connection ids is a *distinct* `(pm_connection_id, id)` pair, so pair-keying
will not collapse it.

**Failure scenario:** Corpid `acme` has rows for locations 1 and 2, both provisioned with PTR id 4321
as the migration requires. The loop issues `AssignedToUserID,eq,4321` twice against
`acme.api.rentmanager.com`. Unless RM's `X-RM12Api-LocationID` header narrows `ServiceManagerIssues`
— which this codebase has not validated for issues; `SystemTokenContext`'s docblock only claims live
validation for location-bound resources (Tenants, Vendors) — every PTR task comes back twice with a
different `pm_connection_id`, `total` doubles, and the duplicate survives scheduling because
`rejectScheduledTasks()` is keyed per `(internal_user_id, pm_connection_id)`: scheduling the copy
tagged location 1 leaves the location-2 copy in the list.

**Suggested fix:** Fan out over one connection per corpid — group the result by `rm_corpid` and take
a stable row per group (the `firstForCorpid()` ordering is the established tiebreak) — or dedupe the
merged rows on `(rm_corpid, id)`. Either way, add a test with two active rows sharing a corpid; the
suite today only covers two *different* corpids (`testcorp` / `othercorp`). If the intent is that the
Location header does scope issues per location, that deserves a one-line note in the docblock,
because it is the assumption the whole loop rests on.

**Comment for the dev:** One thing the tests don't cover: two active rows of the *same* corpid. The
migration in this PR requires the same PTR id on every row of a corpid, and
`forCorpidAndLocation()`'s docblock says multi-location access is real — so the loop hits that
corpid's RM DB once per location with the same `AssignedToUserID` filter. If `X-RM12Api-LocationID`
doesn't narrow `ServiceManagerIssues` (only Tenants/Vendors are documented as validated for that
header), each task comes back duplicated with a different `pm_connection_id`, and the pair-keyed
dedup note doesn't catch it since the pairs genuinely differ. Grouping by `rm_corpid` before the
fan-out, plus a test with two rows sharing a corpid, would settle it either way.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
        if ($connections->isEmpty()) {
            // Nothing provisions sa_ptr_rm_user_id automatically yet, so this
            // is the state of every connection until it's set by hand — make
            // that diagnosable rather than silently indistinguishable from
            // "no pending tasks", the exact bug this ticket reported.
            Log::warning('task_service.pending.no_ptr_connections_provisioned');
```

**Severity:** `should-fix`

**What's wrong:** The warning only fires when *no* active connection has a PTR id. A partially
provisioned estate — some active connections still null — is silently narrowed with nothing in the
logs.

**Why:** The all-or-nothing case is the easy one to notice; the partial one is the expensive one, and
`RULES.md` → DEVSYM-399 as-built names it explicitly: *"A half-provisioned corpid fails on ONE
location only ... That asymmetry reads like a data problem in RM, not a missing column value, and is
expensive to debug."* Same section as the general rule this PR already satisfies elsewhere: *"No
silent caps — a partial result returned because the 'impossible' case happened must be diagnosable,
not invisible."* Since the migration states provisioning is manual and must cover every row of every
corpid, partial state is the *likely* state, not the edge case.

**Failure scenario:** Five active connections, four provisioned by hand, the fifth missed. The
superuser's planner silently omits that PM's tasks. Logs are clean, so the only signal is a PM
reporting missing work — indistinguishable from "that PM has no pending tasks", which is the
ambiguity this ticket was filed as.

**Suggested fix:** Count `PmConnection::query()->active()->whereNull('sa_ptr_rm_user_id')` alongside
the existing query and `Log::warning` with those connection ids when the count is greater than zero,
keeping the existing empty-case warning as-is. Payload unchanged.

**Comment for the dev:** The `no_ptr_connections_provisioned` warning covers "none provisioned", but
not "some". Given the column is provisioned by hand per row, a missed row is the likely state, and
`RULES.md` calls that asymmetry out specifically as expensive to debug. Could we also warn with the
ids of active connections that still have a null `sa_ptr_rm_user_id`? Same payload, just makes the
partial case visible.

---

### Nits

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
        $actingConnection = app(ActingPmConnection::class);
```
```php
                foreach ($connections as $connection) {
                    $actingConnection->set($connection);
```

**Severity:** `nit`

**What's wrong:** The request-scoped acting connection is overwritten inside the loop and never
restored — and only on a cache miss, so the request's acting connection after `pending()` depends on
cache state.

**Why:** `ActingPmConnection`'s docblock: *"Set once per request by form requests that validate an
explicit `pm_connection_id` (see `ResolvesActingPmConnection`)"*. This is the first writer outside
that trait. Nothing downstream of `pending()` reads it today, so there is no live bug — but it turns
a request-scoped invariant into something conditional on whether Redis had the key.

**Suggested fix:** Capture `$actingConnection->get()` before the loop and restore it in a `finally`.
Note `set()` only takes a non-null `PmConnection`, so restoring a genuinely-null starting state needs
a `clear()` or a nullable setter. If the hand-off is deliberate, a line in the docblock saying so is
enough.

**Comment for the dev:** Tiny one — the loop leaves `ActingPmConnection` pointing at the last
connection, and only when the cache misses, so the post-`pending()` state differs between a hit and a
miss. Nothing reads it after this call today, so no bug; restoring the previous value in a `finally`
(or just noting the hand-off is intentional in the docblock) would keep the "set once per request"
invariant honest. `set()` isn't nullable, so a true null restore would need a `clear()`.

---

## Verified clean (no finding)
- Layer discipline holds: no PascalCase RM field outside `app/Modules/Integrations/RentManager/`; the
  RM filter is built via `RentManagerIssueMapper::withAssignedToFilter()` / `listParams()`, and
  `pm_connection_id` / `sa_ptr_rm_user_id` are SA snake_case.
- The per-corpid status-catalog resolution is correctly *inside* the loop, after `set()` — the
  DEVSYM-461 trap the comment cites is genuinely avoided, and
  `test_pending_merges_ptr_tasks_across_every_connection_with_a_ptr_user_id` proves it with two
  different `ServiceManagerStatusID` sets per corpid.
- All three round-1 findings addressed (see the status section above).
- `test_pending_does_not_warn_about_ptr_provisioning_for_a_pm_user()` correctly guards against
  misattributing the warning to a PM with a null `rm_user_id`.
- The `Http::fake()` boundary choice is right: one `Http::assertSent` covers corpid routing, Partner
  Token header, and assignee filter together, which mocking `RentManagerService` could not.
- The inactive third connection in the merge test plus `Http::assertNotSent(... 'inactivecorp')` pins
  the `active()` scope.
- `PmConnection` model change is minimal and correct: docblock `@property`, `$fillable`, no cast
  needed for an int column.
- Migration follows the `2026_07_25_000002` precedent — new timestamped file, `unsignedInteger`,
  nullable, reversible `down()`.
- `pending()`'s PM branch is unchanged in behaviour: the `Auth::user()` hoist to `$user` is a pure
  refactor, `(int) ($user->rm_user_id ?? 0)` is byte-equivalent to the original.
- `Log` and `Cache` were already imported; no unused imports added.
- `$data->refresh` handling is correct: `Cache::forget($cacheKey)` before `Cache::remember`, and
  `fetchList()` itself does not cache, so there is no second stale layer underneath.

## Verdict
needs changes before merge

## SPACE log
- Time spent reviewing: 45 min (Johnny-provided, 2026-08-04)
- Model: Opus
- Findings: 2 blockers, 3 should-fix, 1 nit
- Verdict: needs changes before merge

---

# Round 3 — re-review of the dev's fixes

## Review opened
- Timestamp: 2026-08-04
- Branch: `fix/devsym-517-superuser-daily-planner-ptr-tasks` (`c43c53c`)
- Base branch: `dev`
- New commit under review: `c43c53c fix(DEVSYM-517): warn on partial PTR provisioning, isolate connection failures, fix cache/sort ordering`

## Files in scope
Unchanged file set; totals grew to 646 insertions, 3 deletions.
```
app/Models/PmConnection.php
app/Modules/Tasks/Services/TaskService.php
database/migrations/2026_07_31_000001_add_sa_ptr_rm_user_id_to_pm_connections_table.php
tests/Feature/Modules/InternalUsers/InternalUserPmContextEndpointsTest.php
```

## Test run (new this round — `vendor/` and Docker were available)
```
docker compose exec -T app php artisan test --filter=InternalUserPmContextEndpointsTest
Tests: 1 failed, 37 passed (130 assertions)
```

## Round-2 findings — status
- **blocker (per-user filter inside an actor-free cache)** → FIXED, correctly and as suggested. The cache now holds only raw merged rows (`$rawTasks`); `rejectScheduledTasks()` runs post-cache, grouped by `pm_connection_id`, re-pointing `ActingPmConnection` per group so the per-corpid status whitelist is still applied per corpid. The docblock now states the actor-free key is deliberate (shared bucket) and explains why the filter must stay outside. Pinned by `test_pending_does_not_leak_one_superusers_schedule_filter_to_another()`, which is the right shape: userA schedules → disappears for userA, still present for userB.
- **blocker (no per-connection isolation)** → FIXED. `try/catch (\Throwable)` around the RM work, sanitized log (`pmConnectionId` + `exceptionClass` + `exceptionMessage` only), `continue`. Mirrors `CrossPmFanout::fanOut()` as asked. Pinned by `test_pending_skips_a_connection_that_throws_instead_of_failing_the_whole_request()`. See round-3 should-fix below for the remaining caching consequence.
- **should-fix (slice before sort)** → FIXED. `sortByDueDateAscending()` now runs on the full filtered array, `array_slice()` after. Pinned by `test_pending_sorts_the_full_merge_before_paginating()` with `per_page=1` and the due-sooner task on the second-queried connection — a genuinely regression-proof test.
- **should-fix (partial provisioning silent)** → FIXED. New `task_service.pending.ptr_user_id_not_provisioned_for_some_connections` warning carrying the connection ids. Pinned by `test_pending_warns_with_the_ids_of_connections_still_missing_a_ptr_user_id()`.
- **should-fix (same-corpid multi-location duplication)** → NOT ADDRESSED. No grouping by `rm_corpid`, no dedup, no docblock note, no test with two active rows sharing a corpid. Carried forward unchanged below.
- **nit (`ActingPmConnection` never restored)** → NOT ADDRESSED, partially improved. The post-cache filter loop now calls `set()` on every request rather than only on a cache miss, so the hit/miss inconsistency is gone; the acting connection is still left pointing at the last connection with rows and never restored.

## Findings

### Blockers

---

**File:** `tests/Feature/Modules/InternalUsers/InternalUserPmContextEndpointsTest.php`

**Code snippet:**
```php
        $connectionOne = $this->pmConnection(ptrRmUserId: 111, corpid: 'testcorp');
        $connectionTwo = $this->pmConnection(ptrRmUserId: 222, corpid: 'othercorp');
        PmConnection::create([
            'rm_corpid' => 'inactivecorp',
            'rm_location_id' => 5,
            'status' => 'disconnected',
            'sa_ptr_rm_user_id' => 333,
        ]);
```

**Severity:** `blocker`

**What's wrong:** `test_pending_merges_ptr_tasks_across_every_connection_with_a_ptr_user_id()` does not pass — `'disconnected'` is not a legal `pm_connections.status` value, so the row insert aborts before any assertion runs.

**Why:** `2026_07_01_000001_create_pm_connections_table.php` declares
`$table->enum('status', ['pending', 'active', 'suspended'])->default('pending')`, and no later
migration alters it (checked all 13 migrations touching `pm_connections`). Postgres rejects the
insert with `SQLSTATE[23514] Check violation ... violates check constraint
"pm_connections_status_check"`. The code-review skill is explicit: *"Do all existing tests still
pass? (`php artisan test`)"*. This test is new in this branch, so the failure is introduced by
the PR, not inherited from `dev`.

**Failure scenario:** `docker compose exec app php artisan test --filter=InternalUserPmContextEndpointsTest`
→ `1 failed, 37 passed`, failing at `InternalUserPmContextEndpointsTest.php:670` with the check
violation above. The test's actual subject — that the `active()` scope excludes non-active
connections from the fan-out — is never exercised, so that scope is currently unverified.

**Suggested fix:** Use a legal non-active value. `'pending'` is the convention already used for
exactly this purpose at line 178 of this same file
(`PmConnection::create(['rm_corpid' => 'inactivecorp', 'rm_location_id' => 9, 'status' => 'pending'])`);
`'suspended'` is used elsewhere in the suite. Then re-run the file.

**Comment for the dev:** `test_pending_merges_ptr_tasks_across_every_connection_with_a_ptr_user_id`
is failing — `'disconnected'` isn't a legal `pm_connections.status`; the enum is
`pending|active|suspended` (`2026_07_01_000001`), so Postgres rejects the insert with a check
violation at line 670 and the test never reaches its assertions. Line 178 of this same file
already does this case with `'status' => 'pending'` — same fix here. Worth flagging that this
means the `active()`-scope exclusion is currently untested, which is a shame because it's the
part that test was added for.

**Reviewer note (mine, not the dev's):** I marked this test "verified clean" in round 2 without
running it. That was wrong — it has been failing since `b494c5f`, when it was written.
`vendor/` and the Docker stack were available this round, so the suite is now actually executed.
Going forward: run the suite before writing a "verified clean" line about a test.

---

### Should-fix

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
                    } catch (\Throwable $e) {
                        // Sanitized log: class + message only, never the full
                        // Throwable, which may include RM response bodies.
                        Log::warning('task_service.pending.connection_skipped', [
                            'pmConnectionId' => $connection->id,
                            'exceptionClass' => $e::class,
                            'exceptionMessage' => $e->getMessage(),
                        ]);

                        continue;
                    }
```
The skip happens inside the cached closure:
```php
        $rawTasks = Cache::remember(
```
under a key that carries no actor by design:
```php
        $cacheKey = 'tasks_list_ptr_v'.TaskCacheVersion::current().'_'.md5((string) json_encode([
```

**Severity:** `should-fix`

**What's wrong:** The `continue` runs inside `Cache::remember`, so a partial fan-out is cached for
the full 5-minute TTL — and because the key deliberately has no actor, that partial result is
served to every superuser.

**Why:** The isolation fix is correct in itself, but it converts a hard failure into a cached
silent omission. `RULES.md` → Superuser stack ownership: *"Any aggregation built on
`CrossPmFanout` must surface which connections were reached vs skipped alongside its counts ...
The fan-out's per-item isolation drops broken connections silently (logs server-side only), so
counts alone cannot distinguish 'PM empty' from 'PM dropped'."* This fan-out is hand-rolled
rather than built on `CrossPmFanout`, so the rule's literal scope does not cover it — but it is
the same failure mode the rule exists for, made longer-lived by the cache. `DEVSYM-374`'s
`PortfolioCacheService` satisfies it via `meta.connections.{reachable,reached,skipped}`.

**Failure scenario:** Three provisioned corpids. Corpid B's RM returns a transient 500 (or times
out) on one request. The merge is cached without B's tasks. RM recovers two seconds later, but
for the next five minutes every superuser's planner is missing corpid B's tasks, with a clean
`200` and a `total` that silently excludes them. Only a server-side log line and `?refresh=1`
reveal it.

**Suggested fix:** Either skip the cache write when at least one connection was skipped (return
the partial result for this request only), or carry the skipped connection ids through the cached
payload and surface them in `meta` — mirroring `PortfolioCacheService`'s
`meta.connections.skipped` so the FE and QA can distinguish "PM empty" from "PM dropped". The
first option is the smaller change and matches the "no silent caps" rule; the second matches the
374 precedent.

**Comment for the dev:** The isolation is right, but the `continue` sits inside
`Cache::remember`, so a partial merge gets cached for the whole TTL — and since the key has no
actor (deliberately, which I agree with), every superuser gets that partial view. A transient
500 on one corpid therefore hides that PM's tasks for five minutes after RM has already
recovered, behind a clean `200`. Simplest fix is to not write the cache when any connection was
skipped; the fuller one is to carry the skipped ids into `meta`, like
`PortfolioCacheService`'s `meta.connections.skipped` does for the 374 fan-out. Either is fine by
me — the current shape is the one `RULES.md` warns about, just cached.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
        $connections = PmConnection::query()->active()->whereNotNull('sa_ptr_rm_user_id')->get();
```

**Severity:** `should-fix`

**What's wrong:** Carried forward from round 2, unchanged. The fan-out iterates connection *rows*,
but the PTR id is a corpid-level value replicated across every row of that corpid, so a
multi-location corpid is queried once per location with an identical assignee filter and nothing
dedupes the result.

**Why:** Unchanged from round 2 — `RULES.md` → DEVSYM-399 as-built (*"one row per (corpid,
location) ... must be written to EVERY row of that corpid"*), the migration in this PR mandating
the same id on every row, and `PmConnection::forCorpidAndLocation()`'s DEVSYM-503 docblock
confirming multi-location access is real. The docblock's pair-keyed dedup note does not cover
it: same corpid with two connection ids is a distinct pair.

**Failure scenario:** Unchanged from round 2. Corpid `acme` with rows for locations 1 and 2, both
provisioned with PTR id 4321 as the migration requires → `AssignedToUserID,eq,4321` issued twice
against `acme.api.rentmanager.com`. Unless `X-RM12Api-LocationID` narrows `ServiceManagerIssues`
(unvalidated for issues in this codebase — `SystemTokenContext`'s docblock only claims live
validation for Tenants/Vendors), every PTR task returns twice with a different
`pm_connection_id`, `total` doubles, and the duplicate survives scheduling because
`rejectScheduledTasks()` is keyed per `(internal_user_id, pm_connection_id)`.

**Suggested fix:** Group by `rm_corpid` and fan out one stable row per corpid (the
`firstForCorpid()` ordering is the established tiebreak), or dedupe the merged rows on
`(rm_corpid, id)`. Add a test with two active rows sharing a corpid — the suite still only covers
two *different* corpids. If the intent is that the Location header does scope issues, a one-line
docblock note is enough, since that assumption is what the whole loop rests on.

**Comment for the dev:** This one's still open from the last round, and it's the last real
correctness question here. Two active rows of the *same* corpid — which the migration in this PR
requires to carry the same PTR id, and which `forCorpidAndLocation()`'s docblock says happens for
real — get queried separately with the same `AssignedToUserID` filter. If
`X-RM12Api-LocationID` doesn't narrow `ServiceManagerIssues` (only Tenants/Vendors are documented
as validated for that header), you get every task twice under different `pm_connection_id`s, and
the pair-keyed dedup note doesn't catch it because the pairs genuinely differ. Grouping by
`rm_corpid` before the fan-out plus a test with two rows sharing a corpid settles it either way —
and if you've confirmed the header does scope issues, just say so in the docblock and I'll drop
this.

---

### Nits

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
        $missingPtrConnectionIds = PmConnection::query()->active()->whereNull('sa_ptr_rm_user_id')->pluck('id')->all();
        if ($missingPtrConnectionIds !== []) {
```

**Severity:** `nit`

**What's wrong:** The warning fires on every planner request while the condition holds, and when
nothing is provisioned at all both warnings fire for the same state.

**Why:** The condition is persistent configuration, not an event — the migration documents
provisioning as manual, so "some connections missing" is the steady state until a provisioner
lands, and this logs a warning plus an extra query per request for as long as that is true. The
test asserting `Log::shouldHaveReceived('warning')->twice()` documents the double-warning
deliberately, which is honest, but two lines for one state still reads as duplication to whoever
greps the logs. The information itself is worth having — this is about volume, not the fix.

**Failure scenario:** Superusers refresh the planner through the day with one unprovisioned
connection → thousands of identical `ptr_user_id_not_provisioned_for_some_connections` lines,
which is how a genuinely new warning gets missed.

**Suggested fix:** Cheapest is to fold the two into one warning (the empty case is just "all
connections missing"). If the volume matters, throttle with a short-lived cache key or
`Log::warning` only when the id set changes — but that is more machinery than this probably
deserves before the provisioner ticket lands.

**Comment for the dev:** Non-blocking — the warning is per-request for what is really a
persistent config state, so it'll be a steady stream until the provisioner ticket lands, and the
all-missing case emits two lines for one condition. Folding them into a single warning would
read better in the logs. Your `twice()` assertion documents the current behaviour clearly, so
this is purely a taste call.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```php
            $actingConnection->set($connection);
            $result = $this->rejectScheduledTasks(['data' => $rows]);
```

**Severity:** `nit`

**What's wrong:** Carried forward from round 2. The request-scoped acting connection is still left
pointing at the last connection with rows and never restored — though it now happens consistently
on every request rather than only on a cache miss, which removes the hit/miss divergence I
flagged.

**Why:** `ActingPmConnection`'s docblock: *"Set once per request by form requests that validate an
explicit `pm_connection_id` (see `ResolvesActingPmConnection`)"*. Nothing downstream of
`pending()` reads it, so there is still no live bug.

**Suggested fix:** Capture `$actingConnection->get()` before the loops and restore in a `finally`,
or note in the docblock that the hand-off is deliberate. `set()` is not nullable, so a true null
restore needs a `clear()`.

**Comment for the dev:** Still open from last round, still just a nit — and the cache-dependent
part is gone now that the filter loop runs every request, so it bothers me less. A `finally`
restore or a one-line docblock note that the hand-off is intentional would close it.

---

## Verified clean (round 3)
- The post-cache regrouping is correct: `$rowsByConnection` keys on `pm_connection_id` (always set
  by the closure), the filter loop iterates `$connections` so ordering is deterministic, and
  connections with no rows are skipped before `set()`.
- `meta.total` and `data` stay consistent — both derive from `$sorted` after filtering.
- `$data->refresh` still works: `Cache::forget($cacheKey)` precedes `Cache::remember`, and
  `fetchList()` does not cache underneath.
- `try` correctly excludes `$actingConnection->set()` (which cannot throw) and includes the status
  catalog resolution, so a per-corpid catalog failure is isolated too.
- The four new tests each pin exactly one round-2 finding, and
  `test_pending_sorts_the_full_merge_before_paginating()` is built so the old bug would reliably
  fail it (`per_page=1`, due-sooner task on the second-queried connection) rather than passing by
  luck.
- `test_pending_skips_a_connection_that_throws...` correctly uses `withArgs` instead of a total
  `warning()` count, because `RentManagerHttpClient` logs its own warning for the failed response.
- The 37 other tests in the file pass, including the untouched PM-session regression cases.

## Verdict (round 3)
needs changes before merge — 1 blocker (failing test), 2 should-fix (cached partial fan-out;
same-corpid duplication, carried forward), 2 nits.

## Environment note
Mid-review, the working tree was switched off this branch by activity outside this review
(reflog: `checkout: moving from fix/devsym-517-... to chore/phpstan-baseline-voiceai-unused-constant`,
then a cherry-pick). All diffs, the test run, and every snippet above were taken while the tree
was still at `c43c53c`; the remaining verification used `git show c43c53c:<path>`. The branch
itself is intact (`origin/fix/devsym-517-... == c43c53c`) and the tree was left as found — not
switched back.

## SPACE log (round 3)
- Time spent reviewing: TBD
- Model: Opus
- Findings: 1 blocker, 2 should-fix, 2 nits
- Verdict: needs changes before merge

## Status
OPEN — awaiting dev fixes. Not closed (Step 5 pending): no approval/merge confirmation yet.

---

# Round 4 — independent re-pass on the SAME commit (no new dev work)

## Review opened
- Timestamp: 2026-08-04 17:53 -03
- Repo: BE (`SymAssist-Backend`)
- Branch: `fix/devsym-517-superuser-daily-planner-ptr-tasks` (`c43c53c`)
- Base branch: `dev`
- Issue: DEVSYM-517 — PR [#313](https://github.com/Braintly/SymAssist-Backend/pull/313)

**Why this round exists:** the review request named only repo + branch, with no mention
of rounds 1–3. `c43c53c` is the same HEAD round 3 already reviewed — the dev has pushed
nothing since. So this is a blind independent pass over identical code, not a re-review
of fixes. Useful only as calibration plus the handful of angles rounds 1–3 did not cover.
Round 3's findings all stand unchanged.

## Dev's stated focus
Not supplied again. Taken from the PR #313 description, which is now **stale on two
counts** (see round-4 should-fix #2): it still says the multi-corpid fan-out is a
follow-up (`b494c5f`/`c43c53c` ship it) and that the response shape is unchanged (tasks
now carry `pm_connection_id`).

## Files in scope
Identical to rounds 2–3.
```
app/Models/PmConnection.php
app/Modules/Tasks/Services/TaskService.php
database/migrations/2026_07_31_000001_add_sa_ptr_rm_user_id_to_pm_connections_table.php
tests/Feature/Modules/InternalUsers/InternalUserPmContextEndpointsTest.php
```

## Calibration — what this blind pass MISSED
Recorded deliberately, because it is the useful output of a duplicate review.

- **Round 3's blocker (failing test, `'disconnected'` is not a legal `status`) — MISSED.**
  `vendor/` is not installed in this checkout and `AGENTS.md` forbids installing tooling
  on the host, so no suite was run. Confirmed statically after the fact:
  `2026_07_01_000001_create_pm_connections_table.php:16` declares
  `$table->enum('status', ['pending', 'active', 'suspended'])->default('pending')`, and
  none of the other four `pm_connections` migrations alters it — so the insert dies on
  the check constraint. Round 3's finding is correct and still open.
  **Lesson, same one round 3 already wrote about itself:** a review that cannot execute
  the suite must say so up front and must not imply test health either way.
- **Round 2/3's same-corpid multi-location duplication should-fix — MISSED.** Confirmed
  structurally possible: `2026_07_20_203738` replaced `unique(['rm_location_id'])` with
  `unique(['rm_corpid', 'rm_location_id'])`, so two active rows of one corpid differing
  only by location are legal — exactly the shape the migration in this PR requires to
  carry the same PTR id. Still open, still untested.
- Round 3's warning-volume nit and `ActingPmConnection`-not-restored nit were both found
  independently here, which is a small positive signal on those two.

## Findings (new in round 4 only — rounds 1–3 carry forward as written)

### Should-fix

---

**1 — Merged pagination never offsets by `$page`; `total` advertises unreachable rows**

**File:** `app/Modules/Tasks/Services/TaskService.php`
```php
        $sorted = $this->sortByDueDateAscending(['data' => $filtered])['data'];

        return [
            'data' => $perPage !== null ? array_slice($sorted, 0, $perPage) : $sorted,
            'meta' => [
                'pagination' => [
                    'current_page' => $page ?? RentManagerIssueMapper::FIRST_PAGE_NUMBER,
                    'per_page' => $perPage ?? count($sorted),
                    'total' => count($sorted),
                ],
            ],
        ];
```
**What's wrong:** Distinct from round 2's slice-before-sort finding (which was fixed).
The slice always takes the head of the merge (`0, $perPage`) and never offsets by
`$page`, while `total` reports the full merged count — so with an explicit `per_page`
and N provisioned connections, `(N-1) * per_page` rows of the merge are unreachable at
any page.

**Why:** `$data->page` is forwarded into each connection's own `listParams()`, so page 2
fetches RM page 2 *per corpid* rather than continuing the merged page-1 list. `meta.total`
then promises rows no page can return. The docblock defends the shape as mirroring
`mergeTaskLists()`'s acknowledged imperfection, but that precedent over-fetches at most
2× within ONE corpid with dedup; here the multiplier is the number of onboarded PMs and
grows with each one. `RULES.md` → *Code quality*: "No silent caps — a partial result
returned because the 'impossible' case happened must be diagnosable, not invisible."

**Failure scenario:** Three provisioned connections, `per_page=20`. RM page 1 per corpid
→ 60 merged rows; the response returns 20 and reports `total: 60`. The FE renders three
pages, but page 2 re-queries RM page 2 of every corpid — a different row set entirely.
Rows 21–60 of the page-1 merge are unreachable at any page. Inert today (one connection
provisioned by hand; `per_page` has no server-side default, so an omitted `per_page`
returns everything) — hence should-fix, not blocker.

**Suggested fix:** Offset the slice by the requested page over the merged set and fetch
accordingly from each connection, or cap `total` to what is reachable and log the dropped
count with the connection ids. Decide the semantics before coding — this is a contract
question, not a one-liner.

**Comment for the dev:** Different from the slice-before-sort thing you already fixed:
`array_slice($sorted, 0, $perPage)` never offsets by `$page`, but `total` reports the
full merged count. With 3 provisioned connections and `per_page=20` the response
advertises 60 tasks while rows 21–60 of the merge can't be reached at any page — page 2
asks RM for page 2 of each corpid instead of continuing the merged list. Inert with one
connection today, but it scales the wrong way and it's the "task silently missing from
the planner" symptom this ticket exists to fix. Either offset properly, or make `total`
reflect what's reachable and log the drop.

---

**2 — Response contract change not flagged; PR description contradicts the diff**

**File:** `app/Modules/Tasks/Services/TaskService.php`
```php
                    foreach ($result['data'] as $task) {
                        $task['pm_connection_id'] = $connection->id;
                        $merged[] = $task;
                    }
```
**What's wrong:** `pm_connection_id` is a new field on the task payload, present only for
superuser callers, while the PR description says "The endpoint, its parameters and its
response shape are unchanged."

**Why:** `.atl/skills/code-review/SKILL.md` → *API surface*: "If this PR changes an API
contract — is the breaking change explicitly flagged in the PR description?" The
description is also stale on scope: it lists the multi-corpid fan-out and merge as
"Deliberately not in this PR (follow-up)" while `b494c5f` and `c43c53c` implement it.
`RULES.md` → *Git & PR workflow* already records stale descriptions as a known failure
mode. No `.http` example or OpenApi attribute exists for `/api/tasks/pending` today
(pre-existing gap, not introduced here), so nothing is owed there — but the shape claim
must stop being wrong.

**Suggested fix:** Update the description: the merge as shipped, and `pm_connection_id`
as an additive superuser-only response field, with the motivation
(`ServiceManagerIssueID` is only unique per corpid).

**Comment for the dev:** The description is out of sync with the diff on two points: it
says the multi-corpid merge is a follow-up (`b494c5f`/`c43c53c` ship it) and that the
response shape is unchanged (tasks now carry `pm_connection_id` for superusers). The
field itself is right and well-motivated — it just needs flagging as an additive
contract change so the FE knows it exists.

### Nits

---

**3 — Cache key omits the PTR ids**

**File:** `app/Modules/Tasks/Services/TaskService.php`
```php
        $cacheKey = 'tasks_list_ptr_v'.TaskCacheVersion::current().'_'.md5((string) json_encode([
            $connections->pluck('id')->sort()->values()->all(),
            $data->page, $data->perPage, $data->pageNumber, $data->pageSize,
        ]));
```
Hashes connection ids but not `sa_ptr_rm_user_id`, so correcting a mis-typed PTR id keeps
serving the wrong bucket for the rest of the TTL. First-time provisioning changes the
connection set and dodges this; a *correction* does not — and the migration documents the
column as filled by hand, which makes corrections a first-class flow, not an edge case.
Self-heals in `LIST_TTL_MINUTES` (5) and `?refresh=1` works, hence a nit. Fix: add
`$connections->pluck('sa_ptr_rm_user_id')` to the hashed payload.

---

**4 — Two queries over the same `active()` set**

**File:** `app/Modules/Tasks/Services/TaskService.php`
```php
        $connections = PmConnection::query()->active()->whereNotNull('sa_ptr_rm_user_id')->get();

        $missingPtrConnectionIds = PmConnection::query()->active()->whereNull('sa_ptr_rm_user_id')->pluck('id')->all();
```
`AGENTS.md` → *Code quality* (reuse before creating, no incidental duplication). A single
`->get()` plus `partition()` returns both halves and keeps them from drifting if the
`active()` predicate ever changes. Pairs naturally with round 3's warning-volume nit,
which touches the same two lines.

---

**5 — `pm_connection_id` required but its value ignored for superusers**

**File:** `app/Modules/Tasks/Services/TaskService.php`
```php
        $user = Auth::user();

        if ($user instanceof InternalUser) {
            return $this->pendingForSuperuser($data);
        }
```
`ResolvesActingPmConnection::pmConnectionRules()` keeps `pm_connection_id` required for
an internal user, but the merged bucket ignores its value — a superuser with no PM picked
gets a 422 for a query that does not depend on a PM. Not wrong (the FE picker sends the
header on every call, and a test documents it), but a one-line note at the branch plus a
pointer to DEVSYM-373 stops the next reader from taking the requirement as meaningful
scoping.

---

**6 — `ActingPmConnection` never restored — DUPLICATE of round 3's nit, and round 3 rated
it correctly.** Found independently here and initially written up as should-fix; that was
an over-rating. Round 3's reasoning is the right one: nothing downstream of `pending()`
reads the acting connection today, so there is no live bug — a `finally` restore or a
docblock note that the hand-off is intentional closes it. No new information; see round 3.

## Verified clean (round 4, independent of rounds 1–3)
- Migration column type is consistent with its `sa_*_role_id` siblings
  (`2026_07_25_000002`, `unsignedInteger`) and with `users.rm_user_id`
  (`0001_01_01_000000`, also `unsignedInteger`).
- Migration docblock correctly states the per-corpid value / per-row column replication
  rule from `RULES.md` → *DEVSYM-399 foundation*, including the follow-up provisioner's
  obligation to iterate `candidatesForCorpid()`.
- Per-connection `try/catch (\Throwable)` logs `exceptionClass` + `exceptionMessage` only
  — satisfies the sanitized-fan-out rule in `RULES.md` → *RM Integration*.
- `rmStatusIdsExcludingCompleted()` is resolved inside the loop after
  `$actingConnection->set()`, so the DEVSYM-461 per-corpid status-id trap is genuinely
  avoided; the post-cache filter loop re-points per group for the same reason.
- Only raw rows are cached; `rejectScheduledTasks()` re-runs per request, so one
  superuser's schedule filter cannot leak to another — pinned by
  `test_pending_does_not_leak_one_superusers_schedule_filter_to_another()`.
- No PascalCase RM field names outside `app/Modules/Integrations/RentManager/`; the RM
  filter is built through `RentManagerIssueMapper::withAssignedToFilter()`/`listParams()`.
- `pending()`'s PM branch is behaviour-preserving: hoisting `Auth::user()` to `$user` and
  `(int) ($user->rm_user_id ?? 0)` are equivalent to the original.

## Verdict (round 4)
needs changes before merge — no change from round 3. Round 3's blocker (failing test) and
its two should-fix items are still open on this commit; round 4 adds 2 should-fix and 3
nits from angles not previously covered.

## Environment note (round 4)
Pint and PHPStan were NOT run and no test was executed: `vendor/` is absent from this
checkout (`vendor/bin` empty, and the post-checkout hook reports
`vendor/bin/captainhook: No such file or directory`) and `AGENTS.md` forbids installing
tooling on the host. Every claim above is from reading the diff and the referenced source
files with the Read tool. The PR's own linter/test claims are unverified by this round.
The working tree was left on `fix/devsym-517-superuser-daily-planner-ptr-tasks` at
`c43c53c`; no file in `SymAssist-Backend/` was modified.

## SPACE log (round 4)
- Time spent reviewing: 120 min — Johnny-provided 2026-08-04, reported as the CUMULATIVE
  total across every round on this ticket, not round 4 alone. Not broken down per round.
  Round 2 separately recorded 45 min; whether that sits inside this 120 was not confirmed,
  so do NOT sum them for the weekly report — use 120 as the ticket total.
- Model: Opus
- Findings: 0 blockers, 2 should-fix, 3 nits (all new); round 3's 1 blocker + 2 should-fix
  + 2 nits carried forward unchanged
- Verdict: needs changes before merge

## Handoff to the dev (round 4, 2026-08-04)
Five paste-ready review comments were produced for Johnny, one per finding, each anchored
to its line so the dev can act on them independently:
1. **Blocker** — `InternalUserPmContextEndpointsTest.php:673`, `'disconnected'` → `'pending'`,
   then re-run in Docker (SQLite ignores Laravel's enum CHECK constraints and passes green).
2. `TaskService.php:1006` — partial fan-out cached for the full TTL under an actor-free key.
3. `TaskService.php:930` — same-corpid multi-location duplication (open since round 2).
4. `TaskService.php:1048` — slice never offsets by `$page` while `total` advertises
   unreachable rows.
5. PR body (general comment) — description contradicts the diff on scope and response shape.

Nits were deliberately excluded from the handoff at Johnny's request.
Priority guidance given: #1 and #5 are minutes of work; #3 needs a technical decision first
and is the one still unresolved across three rounds.

## Status
OPEN — awaiting dev fixes on `c43c53c`. Johnny confirmed 2026-08-04 that he expects the dev
to correct these, so Step 5 (close) is NOT done: no approval or merge confirmation yet.
Next action on this file: when the dev pushes, re-review the new commit as round 5 (append,
do not overwrite) and ask Johnny for his reading/validation time before closing.
