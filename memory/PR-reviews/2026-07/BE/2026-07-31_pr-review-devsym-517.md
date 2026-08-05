# PR Review — DEVSYM-517 (BE)

## Review opened
- Timestamp: 2026-07-31
- Repo: BE (`SymAssist-Backend`)
- Branch: `fix/devsym-517-superuser-daily-planner-ptr-tasks` (`7454a13`)
- Base branch: `dev` (merge-base `b641fbd`)
- Issue: DEVSYM-517 — [BE] Superuser: Daily Planner no muestra tareas de symassist-PTR

## Dev's stated focus
A superuser's Daily Planner was always empty: the pending list resolved its assignee from
the logged-in RM user, and an internal user has none. It now resolves to the acting PM
connection's `symassist-PTR` user — the identity a superuser actually acts as inside that
PM's Rent Manager account.

- New nullable `pm_connections.sa_ptr_rm_user_id` column holding that corpid's
  `symassist-PTR` RM user id.
- The Daily Planner pending list resolves its assignee per actor: an internal user reads
  the acting connection's PTR id, a PM keeps reading its own RM user id.
- A connection with no PTR id yet returns the existing empty-list response — no RM call,
  no error.
- Feature tests covering both internal-user branches, including proof the assignee
  actually reaches Rent Manager on the right corpid.

Deliberately out of scope (follow-up): multi-corpid fan-out and merge, and the idempotent
command that discovers and persists the PTR id per connection. Today the value is set by
hand.

## Files in scope
```
app/Models/PmConnection.php
app/Modules/Tasks/Services/TaskService.php
database/migrations/2026_07_31_000001_add_sa_ptr_rm_user_id_to_pm_connections_table.php
tests/Feature/Modules/InternalUsers/InternalUserPmContextEndpointsTest.php
```

## Findings

### should-fix

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```
        $data ??= new TaskListData;
        $user = Auth::user();
        // (int) of null is 0, which the guard below already handles. Do NOT
        // rewrite this as `?->sa_ptr_rm_user_id ?? 0` — a nullsafe on the left
        // of ?? trips Larastan's nullsafe.neverNull (RULES.md, DEVSYM-415).
        $rmUserId = $user instanceof InternalUser
            ? (int) app(ActingPmConnection::class)->get()?->sa_ptr_rm_user_id
            : (int) ($user->rm_user_id ?? 0);

        $page = $data->pageNumber ?? $data->page;
        $perPage = $data->pageSize ?? $data->perPage;

        if ($rmUserId <= 0) {
            return [
                'data' => [],
                'meta' => [
                    'pagination' => [
                        'current_page' => $page ?? RentManagerIssueMapper::FIRST_PAGE_NUMBER,
                        'per_page' => $perPage ?? 0,
                        'total' => 0,
                    ],
                ],
            ];
        }
```

**Severity:** `should-fix`

**What's wrong:** An unprovisioned `sa_ptr_rm_user_id` returns an empty planner with no
log — indistinguishable from "this superuser genuinely has no pending tasks".

**Why:** `RULES.md` → Code quality: *"No silent caps — a partial result returned because
the 'impossible' case happened must be diagnosable, not invisible."* This is not a rare
edge here: the migration itself states nothing populates the column yet, so **every**
connection is in this state on merge, and the symptom is byte-identical to the bug the
ticket reports. Without a log there is no way to tell "PTR id missing on connection 7"
from "no tasks" in production.

**Suggested fix:** Hoist the connection into a variable and emit a `Log::warning` in the
internal-user branch only (the `Log` facade is already imported at line 23), carrying
`pm_connection_id` and `rm_corpid`, before falling into the shared `$rmUserId <= 0`
return. Do not change the returned payload.

**Comment for the dev:** The empty-list fallback for a missing `sa_ptr_rm_user_id` is the
right behaviour, but it's currently silent — and since nothing writes the column yet,
that's the state of every connection at merge time. Could we add a `Log::warning` in the
internal-user branch (connection id + corpid) before the `$rmUserId <= 0` return?
Otherwise "PTR id not provisioned" and "no pending tasks" look identical in prod, which is
exactly the ambiguity this ticket was reported as.

---

### nits

**File:** `database/migrations/2026_07_31_000001_add_sa_ptr_rm_user_id_to_pm_connections_table.php`

**Code snippet:**
```
 * Nullable and per-corpid for the same reason as the sa_*_role_id columns
 * (2026_07_25_000002): RM ids are only unique within a corpid, so there is no
 * global value to default to, and null means "not provisioned yet" — the
 * planner treats it as an empty list, never an error.
```
```
        Schema::table('pm_connections', function (Blueprint $table) {
            $table->unsignedInteger('sa_ptr_rm_user_id')->nullable();
        });
```

**Severity:** `nit`

**What's wrong:** The value is corpid-scoped but the column is connection-scoped (one row
per location), and it is set by hand.

**Why:** `PmConnection::candidatesForCorpid()` / `firstForCorpid()` exist precisely because
one corpid can have several location rows. So the same PTR id has to be hand-copied to
every row of that corpid; miss one and a superuser acting for *that* location gets a
silently empty planner while the sibling location works. Column type and nullability are
consistent with `2026_07_25_000002` — no objection there.

**Suggested fix:** Note the multi-row implication in the docblock, and make the follow-up
discovery command write the id to **all** rows of the corpid rather than a single
connection.

**Comment for the dev:** Small doc thing — the docblock says "per-corpid", but the column
is per-connection and a corpid can have several location rows (`candidatesForCorpid`).
Worth spelling out that the manual value has to go on every row of the corpid, and that
the follow-up provisioner should write all of them. A half-provisioned corpid fails on one
location only, which is a nasty thing to debug.

---

**File:** `app/Modules/Tasks/Services/TaskService.php`

**Code snippet:**
```
        // (int) of null is 0, which the guard below already handles. Do NOT
        // rewrite this as `?->sa_ptr_rm_user_id ?? 0` — a nullsafe on the left
        // of ?? trips Larastan's nullsafe.neverNull (RULES.md, DEVSYM-415).
```

**Severity:** `nit`

**What's wrong:** The cited PHPStan constraint doesn't match this expression.

**Why:** `nullsafe.neverNull` fires when the operand on the left of `?->` is **not**
nullable. The DEVSYM-415 case in `RULES.md` was `$user?->password ?? DUMMY_HASH`, where
`$user` had already been narrowed to non-null. Here `ActingPmConnection::get():
?PmConnection` is genuinely nullable — the line already uses `?->` against it and passes
analysis, so adding `?? 0` would not change that. PHPStan could not be run during this
review (no `vendor/` in the checkout), so this is flagged rather than asserted. The
concern is that a directive comment in this codebase becomes institutional memory.

**Suggested fix:** Either confirm with `./vendor/bin/phpstan analyse` and cite the actual
error, or drop the "Do NOT rewrite" sentence and keep only the `(int) null is 0` note.

**Comment for the dev:** Minor — the `nullsafe.neverNull` note doesn't quite apply here:
that rule fires when the left side of `?->` is non-nullable, and `ActingPmConnection::get()`
is genuinely `?PmConnection` (the DEVSYM-415 case had `$user` already narrowed). Could you
double-check with PHPStan? If it doesn't actually complain, I'd trim the "Do NOT rewrite"
line and keep just the `(int) null is 0` explanation.

---

## Verified clean (no finding)
- Layer discipline holds — no PascalCase RM field outside `app/Modules/Integrations/RentManager/`;
  `sa_ptr_rm_user_id` is an SA snake_case column.
- `app(ActingPmConnection::class)->get()` matches the established convention
  (`TaskStatusService`, `TaskStatsService`, `UserService`, `AttentionNeededService`,
  `PropertyPerformanceService`, `AssignsSubtaskUsers`).
- Cache isolation already correct: `cachedFetchList()` keys on `connectionCacheKeySuffix()`,
  so no cross-corpid bleed even if two corpids shared a PTR id value.
- `PmConnectionController::index()` selects explicit columns — the new column does not leak
  into the PM-picker payload.
- `rejectScheduledTasks()` → `scheduledTaskIdsForUser()` → `preferenceOwner()` already
  handles `InternalUser` via `DailyPlannerActingPreference`; the 403 branch there is
  unreachable from `pending()` because a null acting connection short-circuits earlier.
- `isPropertyScopedUser()` is safe for `InternalUser` (no `user_type` attribute → null →
  `in_array(..., true)` false).
- PM regression path covered by the untouched `TaskPendingControllerTest`
  (`test_queries_rm_by_assignee_and_sa_status_whitelist` asserts `AssignedToUserID,eq,393`).
- `test_internal_user_task_with_null_status_id_is_still_rejected` uses a `User`, not an
  `InternalUser` — no test erosion from the new branch.
- `config(['app.dev_auth_bypass' => false])` in `setUp()` satisfies the `BypassAuthForDev`
  rule in `RULES.md`.
- Migration follows the `2026_07_25_000002` precedent: new timestamped migration (never an
  edit to an already-run one), `unsignedInteger`, nullable, reversible `down()`.

## Verdict
needs changes before merge

## SPACE log
- Time spent reviewing: TBD
- Model: Opus
- Findings: 0 blockers, 1 should-fix, 2 nits
- Verdict: needs changes before merge
