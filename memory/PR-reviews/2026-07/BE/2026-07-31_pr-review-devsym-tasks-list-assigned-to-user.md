## Review opened
- Timestamp: 2026-07-31
- Repo: BE
- Branch: feature/devsym-tasks-list-assigned-to-user
- Base branch: dev
- Issue: devsym-tasks-list-assigned-to-user (PR #308)

## Dev's stated focus
`GET /api/tasks` batch-resolves each page's `AssignedToUserID`s via a single Rent Manager `Users` lookup, so the list can surface the assignee's name. Task list, task detail, and subtasks now all emit the same `assigned_to_user: {user_id, username, name, last_name}` shape (previously detail/subtasks lacked `last_name`, list had nothing beyond the bare id). The object is always present, null-filled when there's no resolved assignee.

## Files in scope
```
app/Modules/Integrations/RentManager/Mappers/RentManagerIssueMapper.php
app/Modules/Tasks/Services/TaskService.php
tests/Feature/Modules/Tasks/TaskListControllerTest.php
tests/Unit/Modules/Integrations/RentManager/Mappers/RentManagerIssueMapperTest.php
tests/Unit/Modules/Tasks/Services/SubtaskServiceTest.php
```

## Findings

### Should-fix

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
    private function hydrateAssignedToUsers(array $items): array
    {
        $userIds = [];
        foreach ($items as $item) {
            $userId = (int) ($item['AssignedToUserID'] ?? 0);
            if ($userId > 0) {
                $userIds[] = $userId;
            }
        }

        $uniqueUserIds = array_values(array_unique($userIds));
        if ($uniqueUserIds === []) {
            return $items;
        }

        $users = $this->rentManagerService->getUsers([
            'filters' => 'UserID,in,('.implode(',', $uniqueUserIds).')',
            'fields' => 'UserID,Name,Firstname,Lastname,Username',
            'pageSize' => count($uniqueUserIds),
        ])['data'] ?? [];
```
**What's wrong:** Duplicates `RentManagerIssueService::resolveSubtaskAssignees()`'s batched-lookup logic (unique ID list → `UserID,in,(...)` filter → `getUsers()` → `usersById` map) almost line-for-line instead of extracting a shared helper.
**Why:** code-review skill "No duplication" checklist — dev's own docblock says this "mirrors" the existing method, meaning it was known and copied rather than reused. Two copies will drift independently.
**Suggested fix:** Extract the `getUsers()` → `usersById` construction into a shared RM-layer method (e.g. `RentManagerUserService::resolveUsersById(array $userIds): array`) used by both call sites.
**Comment for the dev:** Nice catch reusing the batching pattern conceptually — but since you already reference `resolveSubtaskAssignees()` in the docblock, could we pull the "IDs → `usersById` map" logic into one shared method instead of having two copies that can drift apart?

### Nit

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
            'fields' => 'UserID,Name,Firstname,Lastname,Username',
```
**What's wrong:** `Firstname` is requested but never consumed — `RentManagerIssueMapper::buildAssignedToUser()` only reads `UserID`, `Username`, `Name`, `Lastname`.
**Why:** Unused fetched field reads as if it's needed somewhere when it isn't.
**Suggested fix:** Drop `Firstname` from the fields string, or comment why it's reserved for near-future use.
**Comment for the dev:** Small one — `Firstname` looks unused in `buildAssignedToUser()`. Worth trimming unless it's for something upcoming.

## Verification done
- Confirmed `toTaskDetailed()` still gets `assigned_to_user` correctly (inherited via `toTask()` call at the top of the method) after the duplicate inline block was removed — not a regression.
- Confirmed `DEFAULT_LIST_FIELDS` (list endpoint) never included `AssignedToUser`, unlike `DETAIL_FIELDS` — so the new batched hydration is genuinely needed for the list, no redundant/conflicting embed.
- Ran `php artisan test --filter="RentManagerIssueMapperTest|TaskListControllerTest|SubtaskServiceTest"` in Docker — 69 passed, 167 assertions.
- Ran `phpstan analyse` on both changed source files — no errors.
- Ran `pint --test` on all 5 changed files — pass.

## Verdict
ready to merge (should-fix is a minor dedup, can be follow-up at Johnny's discretion)

## SPACE log
- Time spent reviewing: 30 min
- Model: Claude Sonnet 5
- Findings: 0 blockers, 1 should-fix, 1 nit
- Verdict: ready to merge

---

## Re-review — fix commit fed268f
- Timestamp: 2026-07-31
- Commit: `fed268f` "fix(tasks): resolve assigned_to_user for the RM UserID-0 sentinel on the list too"
- Files touched by this commit: `app/Modules/Tasks/Services/TaskService.php`, `tests/Feature/Modules/Tasks/TaskListControllerTest.php`

### What it fixes
Unrelated to the two findings above. `hydrateAssignedToUsers()` was skipping `AssignedToUserID <= 0`, treating `0` purely as SA's "unassigned" filter sentinel. On this RM tenant, `UserID 0` is a real, nameable user ("Historic", legacy/orphaned assignments) that the detail endpoint already resolved via RM's own embed — the list now resolves it too via `isset($item['AssignedToUserID'])` instead of `$userId > 0`.

**File:** `app/Modules/Tasks/Services/TaskService.php`
**Code snippet:**
```php
        $userIds = [];
        foreach ($items as $item) {
            if (! isset($item['AssignedToUserID'])) {
                continue;
            }
            $userIds[] = (int) $item['AssignedToUserID'];
        }
```
**Severity:** clean — no finding
**Why it's correct:** `isset()` still skips a genuinely absent/null key (same as the old `?? 0` + `> 0` combo did for that case), but now includes `0` in the batch lookup and in the resolved-injection map. New test `test_returns_the_resolved_assignee_even_when_assigned_to_user_id_is_the_zero_sentinel` asserts `UserID,in,(0)` is sent and the `Historic` name comes back.

### Verification done
- `php artisan test --filter="RentManagerIssueMapperTest|TaskListControllerTest|SubtaskServiceTest"` — 70 passed, 171 assertions.
- `phpstan analyse` on `TaskService.php` — no errors.
- `pint --test` on `TaskService.php` + `TaskListControllerTest.php` — pass.

### Status of prior findings
- should-fix (dedup with `resolveSubtaskAssignees()`) — still open, not touched by this commit.
- nit (unused `Firstname` field) — still open, not touched by this commit.

### Verdict
ready to merge (fix is correct and well-tested; the two prior findings remain open at Johnny's discretion)

### SPACE log (re-review)
- Time spent reviewing: not recorded (superseded before Johnny answered)
- Model: Claude Sonnet 5
- Findings: 0 new blockers, 0 new should-fix, 0 new nits
- Verdict: ready to merge

---

## Re-review — refactor commit 013f194
- Timestamp: 2026-07-31
- Commit: `013f194` "refactor(rent-manager): extract shared ids->usersById batching into RentManagerUserService"
- Files touched by this commit: `RentManagerIssueService.php`, `RentManagerService.php`, `RentManagerUserService.php`, `TaskService.php`, `TaskListControllerTest.php`, `RentManagerIssueServiceTest.php`

### What it does
Directly addresses the prior should-fix. Extracted the "batch UserIDs → one Users lookup → keyed usersById map" logic (previously duplicated in `hydrateAssignedToUsers()`, `resolveSubtaskAssignees()`, and a third, previously-unnoticed copy in `hydrateActivityLogActors()`) into `RentManagerUserService::resolveUsersById(array $userIds): array`, exposed via `RentManagerService::resolveUsersById()`. All three call sites now delegate to it.

### Status of prior findings
- **should-fix (dedup)** — RESOLVED. Three copies (not two) consolidated into one RM-layer method.
- **nit (unused `Firstname`)** — RESOLVED as a side effect: `hydrateActivityLogActors()` (the third, newly-shared caller) genuinely reads `Firstname` (`RentManagerIssueService.php` ~line 355, `needsCreateUserHydration()`'s field-blank check), so the shared `fields` string is now justified.

### New finding

**File:** `app/Modules/Integrations/RentManager/Services/RentManagerUserService.php`
**Code snippet:**
```php
    /**
     * Batch-resolves a set of RM UserIDs to their user records in one Users
     * lookup, keyed by UserID for O(1) lookup by callers. Dedupes internally
     * so callers can pass ids straight off their rows without pre-filtering.
     * Shared by TaskService::hydrateAssignedToUsers() (task list assignees)
     * and RentManagerIssueService::resolveSubtaskAssignees() (subtask
     * assignees) so the "ids -> usersById map" logic lives in one place.
     *
     * @param  int[]  $userIds
     * @return array<int, array<string, mixed>>
     */
    public function resolveUsersById(array $userIds): array
```
**Severity:** `nit`
**What's wrong:** Docblock says "Shared by X and Y" but lists only 2 of the 3 real callers — omits `RentManagerIssueService::hydrateActivityLogActors()`.
**Why:** An incomplete "shared by" list can mislead a future editor into assuming there are only 2 call sites and breaking the third unknowingly.
**Suggested fix:** Add `hydrateActivityLogActors()` to the docblock's caller list.
**Comment for the dev:** Great consolidation — just add `hydrateActivityLogActors()` to the "Shared by" list in the docblock, it's 3 callers now, not 2.

### Verification done
- `php artisan test --filter="RentManagerIssueMapperTest|TaskListControllerTest|SubtaskServiceTest|RentManagerIssueServiceTest"` — 99 passed, 272 assertions.
- `phpstan analyse` on all 4 changed source files — no errors.
- `pint --test` on all 6 changed files — pass.

### Verdict
ready to merge

### SPACE log (re-review 2)
- Time spent reviewing: 40 min
- Model: Claude Sonnet 5
- Findings: 0 blockers, 0 should-fix, 1 nit (docblock caller list)
- Verdict: ready to merge
