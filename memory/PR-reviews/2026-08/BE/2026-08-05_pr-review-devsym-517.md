# PR Review — DEVSYM-517 (BE)

## Review opened
- Timestamp: 2026-08-05
- Repo: BE (`SymAssist-Backend`)
- Branch: `fix/devsym-517-superuser-daily-planner-ptr-tasks`
- Base branch: `dev`
- Issue: DEVSYM-517 (PR #313, OPEN)
- Reviewer model: Opus 5
- Note: this is a **re-review**. PR #313 already went through one round (3 findings addressed, promoted to `RULES.md`), and the branch has since gained 4 more commits that put the multi-corpid fan-out — declared out of scope in the PR body — into this PR.

## Dev's stated focus
From the PR description:

> A superuser's Daily Planner was always empty: the pending list resolved its assignee from the logged-in RM user, and an internal user has none. It now resolves to the acting PM connection's `symassist-PTR` user.

- New nullable `pm_connections.sa_ptr_rm_user_id` holding that corpid's `symassist-PTR` RM user id.
- The Daily Planner pending list resolves its assignee per actor: internal user → acting connection's PTR id; PM → its own `rm_user_id`.
- A connection with no PTR id returns the existing empty-list response and logs a warning.
- Feature tests covering both internal-user branches.

Description explicitly claims: response shape unchanged, and multi-corpid fan-out/merge deliberately deferred to a follow-up. **Both are stale** — see should-fix 4.

Commits on the branch (newest first):
```
aa3dbcf fix(DEVSYM-517): paginate over the full merged PTR bucket, not per-connection pages
30953fc fix(DEVSYM-517): fan out once per corpid, not once per active row
d57e562 fix(DEVSYM-517): do not cache a merge with a skipped connection
ed28f31 fix(DEVSYM-517): use a legal pm_connections.status enum value in a test
c43c53c fix(DEVSYM-517): warn on partial PTR provisioning, isolate connection failures, fix cache/sort ordering
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
984 insertions, 3 deletions.

## Findings

> **Severity revision (same session, after pushback).** Findings 1 and 2 were
> originally graded `should-fix` and were downgraded to `nit` on re-examination.
> The reasoning is recorded inline on each. Net bar to merge: 2 should-fix
> (the `finally` restore, and the PR description), 6 nits.

### should-fix

---

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
            $actingConnection->set($connection);
            $result = $this->rejectScheduledTasks(['data' => $rows]);
            array_push($filtered, ...$result['data']);
        }
```
**Severity:** `should-fix`
**What's wrong:** `ActingPmConnection` is repeatedly re-pointed and never restored, so after `pending()` returns the request-scoped holder points at the last connection in the merge instead of the caller's `X-Pm-Connection-Id`.
**Why:** `ActingPmConnection`'s own docblock defines the invariant: *"Set once per request by form requests that validate an explicit `pm_connection_id` (see ResolvesActingPmConnection)"*. Both new loops break it. Nothing reads the holder after `TaskController::pending()` today (the controller only does `response()->json($tasks)`), so this is latent rather than broken — but it is the single object `UserSessionContext`, `TaskStatusService::fetchRmStatuses()`, `connectionCacheKeySuffix()` and `DailyPlannerScheduleService::preferenceOwner()` all resolve through, and the failure mode when it does bite is a cross-tenant read. Kept at should-fix on re-examination specifically because the fix is ~3 lines and the failure class is the worst one available in a multi-tenant read path.
**Suggested fix:** Capture `$original = $actingConnection->get()` before the first `set()` and restore it in a `finally` around both loops.
**Comment for the dev:**
> `ActingPmConnection`'s docblock says it's set once per request by the form request, and both loops here leave it pointed at the last connection in the merge rather than the caller's `X-Pm-Connection-Id`. Nothing reads it after the controller today, so nothing is broken — but it's the object `UserSessionContext`, the per-connection cache suffixes and `preferenceOwner()` all resolve through, and a cross-tenant read is an expensive way to find out. Capturing the original and restoring it in a `finally` would keep the mutation scoped to the fan-out.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
            foreach ($rows as $task) {
                $task['pm_connection_id'] = $connection->id;
                $merged[] = $task;
            }
```
**Severity:** `should-fix`
**What's wrong:** `GET /api/tasks/pending` now returns a new `pm_connection_id` field on every task for internal users, but the PR description states the response shape is unchanged and that the multi-corpid merge is out of scope.
**Why:** The PR body says *"The endpoint, its parameters and its response shape are unchanged"* and *"Deliberately not in this PR (follow-up): multi-corpid fan-out and merge"*. Four commits (`b494c5f`, `30953fc`, `aa3dbcf`, `d57e562`) implemented exactly that fan-out, and the response now carries a field the FE has never seen. `meta.pagination.total` also changed meaning — SA-computed count over the merge, not RM's own total. AGENTS.md → *"A PR that changes any API surface MUST include a Testing Endpoints section"*; code-review skill → *"If this PR changes an API contract — is the breaking change explicitly flagged in the PR description?"*. Side note: `/api/tasks/pending` has no entry in `RESTClient/` at all (pre-existing), but this is the PR that gives it a shape worth pinning. Held at should-fix on re-examination: it costs no code, and it is the one finding that will actually mislead another team — the FE reads this description.
**Suggested fix:** Update the PR description to what the branch actually does now (cross-corpid merge in scope, `pm_connection_id` additive, `meta.pagination` SA-computed), refresh the Testing Endpoints section (section 1's example predates the merge), and add a `pending` block to `RESTClient/tasks.http`.
**Comment for the dev:**
> The description is a round or two behind the branch — it says the multi-corpid fan-out is a follow-up and the response shape is unchanged, but `b494c5f`/`30953fc`/`aa3dbcf` landed the merge and tasks now carry a new `pm_connection_id`, with `meta.pagination.total` computed SA-side rather than coming from RM. Worth refreshing the description and the Testing Endpoints examples so the FE has the new field flagged. `/api/tasks/pending` isn't in `RESTClient/` at all either — good moment to add it.

---

### nits

---

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
            try {
                // Status catalog ids are per-corpid (RM assigns them
                // independently per tenant — the exact DEVSYM-461 trap
                // this ticket's own "known traps" section warns about), so
                // this must resolve AFTER pointing ActingPmConnection at
                // this connection, never once before the loop.
                $statusIds = $this->taskStatusService->rmStatusIdsExcludingCompleted();
                $rows = $this->fetchAllPtrPages($connection, $statusIds);
            } catch (\Throwable $e) {
```
**Severity:** `nit` _(originally graded `should-fix`; downgraded on re-examination — see "Trigger reality" below)_
**What's wrong:** A failed status-catalog fetch cannot reach this `catch`, so `anySkipped` stays `false` and a merge that silently lost a whole corpid's tasks gets cached as complete.
**Trigger reality (why this is a nit, not a should-fix):** it needs `ServiceManagerStatuses` to fail *while* `ServiceManagerIssues` on the same connection succeeds — same credentials, same `RentManagerHttpClient`, so a real RM outage fails both and the `catch` does its job. The catalog is also cached for `RM_CATALOG_CACHE_TTL_MINUTES`, so a success is sticky. And the visible outcome — that corpid contributing zero tasks — is **pre-existing**, not a regression: `pending()`'s PM branch passes the same possibly-empty `statusIds` into `listParams()` and has always behaved this way. Only the "cached as complete" half is new to this PR.
**Why:** `TaskStatusService::fetchRmStatuses()` swallows every `\Throwable` and returns `null`, so `rmStatusIdsExcludingCompleted()` returns `[]` instead of throwing. Downstream: `RentManagerIssueMapper::buildInClause()` returns `null` on an empty array, so the RM query loses its status filter entirely and returns the corpid's **whole** PTR history including Completed — up to `PTR_MAX_PAGES × PTR_FETCH_PAGE_SIZE` = 4,000 rows walked and cached. Then `rejectScheduledTasks()` re-derives the same empty whitelist, `$allowedStatusSet` is `[]`, and every one of those rows is filtered out. Net: that corpid shows zero tasks, the merge is written to a 5-minute actor-less cache, every superuser sees the gap — the exact failure mode the `anySkipped` guard from `d57e562` exists to prevent. Also violates RULES.md → "Any aggregation built on `CrossPmFanout` must surface which connections were reached vs skipped": a degraded connection is neither reached nor reported.
**Suggested fix:** Treat an empty `$statusIds` as an incomplete connection — skip it, set `$anySkipped = true`, log alongside `connection_skipped`. (Or have `TaskStatusService` expose "catalog unavailable" distinct from "catalog empty", but the local guard is smaller.)
**Comment for the dev:**
> The `try/catch` here can't catch a status-catalog failure — `fetchRmStatuses()` catches `\Throwable` itself and returns `null`, so `rmStatusIdsExcludingCompleted()` hands back `[]` rather than throwing. With an empty array `buildInClause()` drops the status clause, so RM returns that corpid's full PTR history (Completed included, up to 4,000 rows across 20 pages), and then `rejectScheduledTasks()` filters all of it out because its whitelist is empty too. The corpid ends up contributing zero rows, `anySkipped` stays `false`, and that partial merge is cached for 5 minutes for every superuser — which is the case the cache guard was added for. Could we treat `$statusIds === []` as a skipped connection here?

---

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
            $reachedEnd = $perPage <= 0 || ($page * $perPage) >= $total;
            $page++;
        } while (! $reachedEnd && $page <= self::PTR_MAX_PAGES);

        if (! $reachedEnd) {
            Log::warning('task_service.pending.ptr_pagination_hit_max_pages', [
```
**Severity:** `nit` _(originally graded `should-fix`; downgraded on re-examination — see "Trigger reality" below)_
**What's wrong:** The RM page walk is never exercised beyond page 1 by any test, and both of its truncation branches still yield a merge that `anySkipped` reports as complete and cacheable.
**Trigger reality (why this is a nit, not a should-fix):** the `do-while` body runs once in every test, so the fetch-and-accumulate path *is* covered; only the continuation branch is not. Reaching page 2 needs 200+ open PTR tasks in one corpid, and nothing provisions `sa_ptr_rm_user_id` yet, so today the real bucket is empty everywhere. The `$perPage <= 0` early stop is also copied verbatim from `PortfolioCacheService::fetchAllPages()` — the established precedent in this codebase — so it is a consistency choice, not an oversight. One test closes this; it does not need to hold the merge.
**Why:** Every `Http::fake()` in the new tests returns a bare array with no pagination meta, so `$perPage` resolves to `0` and `$reachedEnd` is `true` after the first iteration in all nine tests. The loop added in `aa3dbcf` — newest, subtlest commit, and the one the docblock spends the most words on — has no coverage of its termination condition or of the `PTR_MAX_PAGES` cap. An off-by-one in `($page * $perPage) >= $total` would leave the whole suite green. Two consequences also go unguarded: `$perPage <= 0` stops after one page with no log (RULES.md → "No silent caps — a partial result returned because the 'impossible' case happened must be diagnosable, not invisible"; the `PortfolioCacheService::fetchAllPages()` precedent at least documents this branch in its docblock, this one doesn't), and hitting `PTR_MAX_PAGES` logs but still returns `$anySkipped = false`, so a clipped bucket is cached as authoritative.
**Suggested fix:** Add a test whose fake returns `meta.pagination` with `total` > `PTR_FETCH_PAGE_SIZE` and asserts two `ServiceManagerIssues` calls with `pageNumber=1` then `pageNumber=2`, plus one pinning the `PTR_MAX_PAGES` cap. Then make both truncation branches propagate incompleteness so the merge is not cached, and document the `per_page <= 0` branch the way the precedent does.
**Comment for the dev:**
> The multi-page walk isn't covered — all the fakes return a bare array with no `meta.pagination`, so `$perPage` is `0` and the loop exits after one iteration in every test. Worth a case with `total` above `PTR_FETCH_PAGE_SIZE` asserting `pageNumber=1` then `pageNumber=2`, and one pinning the `PTR_MAX_PAGES` cap. Separately: both ways the walk can end early (`per_page <= 0`, and the cap) still return `anySkipped = false`, so a clipped bucket gets cached as if it were complete. `PortfolioCacheService` at least documents the `per_page 0` branch in its docblock — worth mirroring that here too.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
        $missingPtrConnectionIds = PmConnection::query()->active()->whereNull('sa_ptr_rm_user_id')->pluck('id')->all();
        if ($missingPtrConnectionIds !== []) {
```
**Severity:** `nit`
**What's wrong:** Both provisioning warnings fire on every request, including cache hits — and since nothing populates the column yet, that is 100% of superuser planner requests.
**Why:** The warnings exist so an unprovisioned connection stays diagnosable, but the planner is a polled endpoint and these run before the cache check. A `warning` on literally every request is the fastest way to make the line worthless, defeating what `c43c53c` bought.
**Suggested fix:** Throttle — `Cache::add("ptr_warn:{$fingerprint}", true, now()->addMinutes(15))` as the emission guard, keyed on the missing-id set so a newly-missed row still warns immediately.
**Comment for the dev:**
> These two warnings fire before the cache check on every request, and since nothing provisions the column yet that's every superuser planner poll. The diagnosability was the point of adding them — a `Cache::add()` throttle keyed on the missing-id set would keep the signal without the volume.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
    /** Page size used walking a connection's full PTR bucket — see fetchAllPtrPages(). */
    private const PTR_FETCH_PAGE_SIZE = 200;
```
**Severity:** `nit`
**What's wrong:** 200 is small for a walk that always fetches the entire bucket, and 5× below the precedent the code says it mirrors.
**Why:** `PortfolioCacheService::PAGE_SIZE` is `1000`, `RentManagerIssueMapper::DAILY_PLANNER_PAGE_SIZE` is `500`, `TaskListRequest` allows `per_page` up to `1000`. Since `fetchAllPtrPages()` walks to the end regardless, a smaller page only multiplies round-trips — and `fetchList()` issues an extra `Users` hydration call per page via `hydrateAssignedToUsers()`, so it is 2 RM calls per page. A corpid with 1,000 PTR tasks costs 10 calls at 200 instead of 2 at 1,000, serially, in one request, per corpid.
**Suggested fix:** Raise to 1,000 to match `PortfolioCacheService`; adjust `PTR_MAX_PAGES` if you want the same 4,000-row effective ceiling.
**Comment for the dev:**
> Since the walk always drains the bucket, the page size only sets how many round-trips it takes — and `fetchList()` adds a `Users` hydration call per page, so it's 2 RM calls each. `PortfolioCacheService` uses 1000 and `DAILY_PLANNER_PAGE_SIZE` is 500; 200 makes a 1,000-task corpid cost 10 calls instead of 2, serially, per corpid.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
            $actingConnection->set($connection);
            $result = $this->rejectScheduledTasks(['data' => $rows]);
```
**Severity:** `nit`
**What's wrong:** This read path now creates a `daily_planner_acting_preferences` row for every connection in the merge.
**Why:** `rejectScheduledTasks()` → `scheduledTaskIdsForUser()` → `DailyPlannerScheduleService::preferenceOwner()`, which does `DailyPlannerActingPreference::firstOrCreate(['internal_user_id' => ..., 'pm_connection_id' => ...])`. Before this PR that was one row for the acting connection; now a single `GET` writes one per provisioned PM, including ones the superuser never acted for. Idempotent, but a write on a GET, and the table stops meaning "connections this user has acted on".
**Suggested fix:** Give `DailyPlannerScheduleService` a read-only lookup for this path (`firstWhere` instead of `firstOrCreate`, empty set when absent), or pass the already-loaded connection in.
**Comment for the dev:**
> `rejectScheduledTasks()` reaches `preferenceOwner()`, which `firstOrCreate`s a `DailyPlannerActingPreference` — so one `GET /api/tasks/pending` now creates a row per provisioned connection, including PMs this superuser never acted for. Harmless but it's a write on a read path, and the table stops meaning "connections this user has acted on". A read-only lookup for this path would avoid it.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
                        'per_page' => $perPage ?? 0,
```
**Severity:** `nit`
**What's wrong:** Two small inconsistencies in the same method: the empty-list return falls back to `per_page => 0` while the populated return uses `per_page => count($sorted)`, and the two `PmConnection` queries could be one.
**Why:** The populated path matches `mergeTaskLists()`'s convention, the `?? 0` matches `emptyList()`'s. Both defensible in isolation, but the same endpoint reporting `per_page` two ways for the same absent parameter is something the FE has to special-case. Separately, `active()->whereNotNull(...)` and `active()->whereNull(...)` are two round-trips for a partition of one result set.
**Suggested fix:** Pick one `per_page` convention across both returns. Load `PmConnection::query()->active()->get()` once and `partition()` on `sa_ptr_rm_user_id` in memory.
**Comment for the dev:**
> Small consistency thing: the empty return reports `per_page => 0` and the populated one `per_page => count($sorted)` for the same absent parameter — worth picking one. And the two `active()` queries could be a single `->get()` plus a `partition()` on `sa_ptr_rm_user_id`.

---

## What is clean (verified, not flagged)
- Migration: new timestamped file, nullable column, docblock spells out per-corpid value vs per-row column and that the follow-up provisioner must iterate `candidatesForCorpid()`. Matches the RULES.md entry promoted from round 1.
- `oneConnectionPerCorpid()` genuinely replicates `PmConnection::firstForCorpid()`'s tiebreak (real location before pending null, then lowest `rm_location_id`) — verified against the model.
- Per-connection `try/catch (\Throwable)` + sanitized log (`exceptionClass` + `exceptionMessage`, never the full Throwable) follows the `CrossPmFanout::fanOut()` rule in RULES.md.
- Status ids resolved *inside* the loop after `ActingPmConnection` is pointed — correct handling of the DEVSYM-461 per-corpid status trap.
- Caching only the RAW merge and re-running `rejectScheduledTasks()` per request is the right call, and `test_pending_does_not_leak_one_superusers_schedule_filter_to_another` is precisely the test the actor-less cache key needed.
- Layer discipline intact: no PascalCase RM field names outside `Integrations/RentManager/`; `pm_connection_id` is snake_case SA.
- No unit tests needed — every new method is private, covered through feature tests.

## Verdict
**needs changes before merge** — but the bar is small. Two items:
1. Restore `ActingPmConnection` in a `finally` (~3 lines).
2. Update the PR description (no code): the multi-corpid merge IS in this PR, `pm_connection_id` is a new response field, `meta.pagination.total` is SA-computed.

The six nits are follow-up material and should not hold the merge.

## Status (2026-08-05, review still open)
- should-fix #1 (`ActingPmConnection` `finally` restore) — **RESOLVED** in commit
  `47ee319` "restore the caller's acting connection after the PTR fan-out".
  Verified against the diff:
  - `$callerConnection = $actingConnection->get()` is captured BEFORE any
    mutation (right after the holder is resolved), so it holds the value the
    FormRequest set.
  - The `try`/`finally` wraps BOTH mutation sites — the
    `fetchPtrTasksAcrossConnections()` call (which sets per connection
    internally) and the `$filtered` loop.
  - `$sorted` moved inside the `try` and is still read after it: correct, PHP
    has function scope, and with no `catch` there is no path that skips the
    assignment and keeps executing.
  - New regression test `test_pending_restores_the_callers_acting_connection_after_the_fan_out`
    genuinely fails without the fix: the `Http::fake` returns `[]` for
    everything, so the `$filtered` loop `continue`s on every connection and the
    ONLY mutation is the one inside the fan-out, leaving the holder on the
    last-iterated corpid instead of the caller's.
- should-fix #2 (stale PR description) — **STILL OPEN**, and now one commit more
  stale. Body still reads "Deliberately **not** in this PR (follow-up):
  multi-corpid fan-out and merge" and "The endpoint, its parameters and its
  response shape are unchanged". Both false on this branch.
- The 6 nits are follow-up material. One new nit added below (iteration order).

### Not verified by this review
`vendor/` is not installed in this checkout, so the test suite, Pint and PHPStan
were NOT run here. The pass/fail claims in the PR body are the dev's, unverified
on my side.

### New nit (from the fix commit)
`test_pending_restores_the_callers_acting_connection_after_the_fan_out` asserts
the holder ends on `$connectionOne`, which is only meaningful because
`othercorp` happens to be iterated last. The source query
(`PmConnection::query()->active()->whereNotNull('sa_ptr_rm_user_id')->get()`)
has no `orderBy`, so iteration order is whatever the DB returns. If it ever
flips, the last-iterated connection IS the caller's and the test passes even
with the fix reverted — a regression test that silently stops testing. Adding
`->orderBy('id')` to that query makes the order deterministic (and with it the
fan-out order, the `pm_connection_id` tagging and the cache-key iteration).

## Review-process note
This PR had already been through several rounds and the dev had addressed every
prior finding. The first pass of this review over-graded two findings as
`should-fix` that, on re-examination, are `nit`:
- the status-catalog degradation needs `ServiceManagerStatuses` to fail while
  `ServiceManagerIssues` on the SAME connection succeeds, and its visible
  outcome is pre-existing in `pending()`'s PM branch;
- the untested page walk covers its fetch path in every test (the `do-while`
  body always runs once) and its early-stop is copied verbatim from the
  `PortfolioCacheService::fetchAllPages()` precedent.

Lesson for future reviews: before grading a finding `should-fix`, state the
concrete trigger and check (a) whether it is a regression or pre-existing, and
(b) whether the shape is copied from an established precedent. Both are
downgrades. On a PR that has already absorbed multiple review rounds, an
inflated severity has a real cost.

## SPACE log
- Time spent reviewing (Johnny, reading + thinking on the PR): 30 min
- Time spent reading/validating this review: 30 min
- Johnny subtotal: 60 min
- Model: Opus 5
- Findings (final): 0 blockers, 2 should-fix, 6 nits
- Findings (first pass, before severity revision): 0 blockers, 4 should-fix, 4 nits
- Verdict: needs changes before merge

## Review closed
- Timestamp: 2026-08-05
- Resolution: approved
- PR: Braintly/SymAssist-Backend#313 (base `dev`), head `fix/devsym-517-superuser-daily-planner-ptr-tasks` @ `47ee319`

### Addressed by the dev during the review
- should-fix #1 — `ActingPmConnection` left pointing at the last connection in
  the merge. Fixed in `47ee319`: caller connection captured before any mutation,
  `try`/`finally` around both mutation sites, plus a regression test that fails
  without the fix. Verified against the diff, not taken on trust.

### Open at approval time (agreed, not waived)
- should-fix #2 — PR description is stale: still claims the multi-corpid fan-out
  is a follow-up and that the response shape is unchanged, while `pm_connection_id`
  is now a new response field and `meta.pagination.total` is SA-computed.
  Requested as a **pre-merge condition**, not a blocker on the approval.

### Deferred to follow-up (7 nits)
Status-catalog degradation escaping `anySkipped`; RM page walk uncovered beyond
page 1; both provisioning warnings firing on every request; `firstOrCreate` on a
GET path; two queries where one would do; `per_page` fallback inconsistency;
non-deterministic iteration order weakening the new regression test.

### Review-quality note
First pass over-graded 4 findings as should-fix; 2 were downgraded to nit after
Johnny pushed back that the PR had already absorbed several rounds. The gating
rule that came out of it is recorded above and in engram
(`process/pr-review-severity-gating`).
