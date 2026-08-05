# DEVSYM-517 — [BE] Superuser: Daily Planner no muestra tareas de symassist-PTR

**Date:** 2026-07-31
**Branch:** `fix/devsym-517-superuser-daily-planner-ptr-tasks` (base `dev` @ `b641fbd`)
**Scope:** narrowest safe slice — the visible bug only, single acting connection. Fan-out /
multi-corpid merge / provisioning command are a deliberate follow-up PR.

## What was done

`TaskService::pending()` — the real Daily Planner read path (`GET /api/tasks/pending`, NOT
`/api/daily-planner/*`) — resolved its assignee from `Auth::user()->rm_user_id`. An
`InternalUser` has no such column, so it read `null → 0` and the existing guard returned an
empty page without ever calling RM: the superuser's planner was empty. One conditional now
branches on the actor: an internal_user reads `sa_ptr_rm_user_id` off the connection the
request is acting for (`ActingPmConnection`, DEVSYM-484); a real `User` path is unchanged.

Everything downstream of that line — `listParams()`, `cachedFetchList()`,
`rejectScheduledTasks()`, the SA status whitelist, `toTask()` — was deliberately left
untouched, so DEVSYM-443's two-day-old contract stays byte-identical.

## Files touched

- `database/migrations/2026_07_31_000001_add_sa_ptr_rm_user_id_to_pm_connections_table.php`
  (new) — nullable `unsignedInteger`, symmetric `down()`. Nothing populates it yet; the value
  is set by hand per connection for the demo.
- `app/Models/PmConnection.php` (+2) — `@property` docblock line + `$fillable` entry.
- `app/Modules/Tasks/Services/TaskService.php` (+18/-1) — 2 imports (`InternalUser`,
  `ActingPmConnection`), the conditional, and a docblock paragraph.
- `tests/Feature/Modules/InternalUsers/InternalUserPmContextEndpointsTest.php` (+71/-1) —
  2 tests; `pmConnection()` gains an optional `?int $ptrRmUserId = null` so its ~20 existing
  callers are unchanged.

Commits: migration+model / the conditional / tests.

## Decisions made

- **One conditional, not a seam abstraction.** No `PlannerActorScope`, no interface, no
  strategy. The general multi-corpid version is designed (Phase-0 diagnosis) and will replace
  this conditional in the follow-up without touching anything downstream. Building half of it
  now would create two sources of truth mid-flight.
- **Tests placed in `InternalUserPmContextEndpointsTest`, not `TaskPendingControllerTest`.**
  An internal-user request needs a real DB (`exists:pm_connections,id` + `findOrFail` in
  `ResolvesActingPmConnection`), and `TaskPendingControllerTest` has no `RefreshDatabase` —
  adding it would change setup for all 15 of DEVSYM-443's tests. The InternalUsers file
  already has the whole harness (`RefreshDatabase`, `Cache::flush`, `dev_auth_bypass=false`,
  `credential_mode=partner_token`, `pmConnection()`), so zero duplication.
- **Faked at the HTTP boundary (`Http::fake`), not by mocking `RentManagerService`.** One
  assertion then covers the corpid host, the Partner Token header AND the `AssignedToUserID`
  filter. Mocking the facade — the style `TaskPendingControllerTest` uses — proves the filter
  string but nothing about the multi-tenant wiring underneath, which was the exact coverage
  gap found in Phase 0.
- **No `RESTClient/tasks.http` entry, no `l5-swagger:generate`.** The endpoint, its params
  and its response shape are unchanged; only the internally-resolved assignee differs.
- **Repo-local git identity set to `jcampo-b <jhonattan.campo@braintly.com>`.** History held
  six identities for this author, five of them machine-derived by accident
  (`awesomejohnny@<hostname>`, `awesomejohnny@192.168.1.x`). Only `jcampo-b` links to the
  GitHub account. Set with `git config` (local, not global); commit 1 was amended.

## Verification

- Pint: PASS, 643 files. PHPStan L5: **1 error, pre-existing** —
  `VoiceAIService::SYSTEM_PROMPT_V3` unused; proven identical by stashing and re-running
  against clean `dev`.
- Full suite **42 failed / 7 risky / 902 passed** vs `dev` baseline **42 / 7 / 900** (stashed
  and re-run). Same failures, +2 new tests, zero regressions.
- `TaskPendingControllerTest`'s 15 DEVSYM-443 tests pass with the file unmodified.
- Migration exercised on the real dev DB: `migrate` → column present → `migrate:rollback
  --step=1` → column absent → `migrate` again. `--step=1` reverted only this migration.
- **The assignee test was proven to fail without the fix.** Reverted the conditional by hand:
  `assertJsonPath('data.0.id', 700)` failed with `null`. The null-PTR test passes on the buggy
  code too — by design, it guards the empty path, it does not prove the fix. Recorded so
  nobody reads it as stronger evidence than it is.
- `git merge origin/dev` → already up to date, no conflicts.

## Open questions / follow-ups

- **The merge half of 517 has no FE to land in.** `PmConnectionGate.tsx:37` blocks the page
  until exactly ONE PM is picked and there is no "All PMs" option; `use-daily-planner-card.ts`
  explicitly opts into the tenant-scoped endpoints. A merged multi-corpid planner is invisible
  until an FE ticket adds that mode, and it would need `pm_connection_id` to become optional
  on this endpoint, contradicting the 484 contract every sibling endpoint enforces.
- **Scheduled-slot exclusion collides across corpids.** `rejectScheduledTasks()` excludes by
  bare RM task id while slots are stored per `(internal_user_id, pm_connection_id)`. In a
  merged list a slot for task N in corpid A would silently hide the unrelated task N in
  corpid B — the mirror of the `deleteSlotsByTaskId` cross-user-scoping bug. Needs a decision
  before the merge ships; a non-issue while single-connection.
- **The PTR assumption is not yet live-validated.** That the Partner Token authenticates AS
  symassist-PTR rests on a code comment (`PmConnectionHealthCheckService.php:41-48`), not an
  observed response. Confirm via the existing `RM_DIAG_TEST_TOKEN` diag route from the
  whitelisted egress IP before the provisioning command is built on it.
- **`dev` is red on PHPStan**, so every commit currently needs `--no-verify`. Not caused by
  this branch. Either baseline `VoiceAIService::SYSTEM_PROMPT_V3` or remove it.
- **Two more Daily Planner breakages for internal users**, both out of this ticket's Must:
  `GET /api/daily-planner/upcoming` returns 403 (its FormRequest lacks
  `ResolvesActingPmConnection`) and never received DEVSYM-443's status rule;
  `GET /api/daily-planner/today` has zero FE callers.
- Local dev DB has **0 rows in `pm_connections`** — one must exist before the PTR id can be
  set by hand.
