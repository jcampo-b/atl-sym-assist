## Review opened
- Timestamp: 2026-07-07
- Repo: BE
- Branch: hotfix/return_status_with_order_ids
- Base branch: dev
- Issue: return_status_with_order_ids (no ticket number in branch)

## Dev's stated focus
Remove useless RRM `sort_order` and add a custom SA `sort_order`. Pending (`<Unassigned>`/Virgin) gets `sort_order = 0`, then 1 to 7 for the rest of the lifecycle statuses. Verified manually via `GET /api/tasks/statuses`.

## Files in scope
```
app/Modules/Integrations/RentManager/Mappers/RentManagerIssueStatusMapper.php
app/Modules/Tasks/Services/TaskStatusService.php
```

## Findings

### should-fix

**File:** `app/Modules/Integrations/RentManager/Mappers/RentManagerIssueStatusMapper.php`
**Code snippet:**
```php
        return array_merge([
            'id' => $id,
            'name' => $status['Name'] ?? null,
            'description' => $status['Description'] ?? null,
            'sort_order' => self::SA_SORT_ORDER[$id] ?? $status['SortOrder'] ?? null,
            'color' => $status['Color'] ?? null,
            'is_active' => $status['IsActive'] ?? null,
        ], $style);
```
**What's wrong:** The fallback to `$status['SortOrder']` is dead code for every status that actually reaches the response.
**Why:** `allowedRmStatusIds()` returns exactly the 8 keys of `RM_TO_LIFECYCLE` (1, 56, 52, 53, 54, 55, 18, 22), and `SA_SORT_ORDER` defines the same 8 keys. `TaskStatusService::list()` filters the mapped array down to only those allowed ids before returning, so any status that survives filtering already has an `SA_SORT_ORDER` entry — the `?? $status['SortOrder']` branch can never surface in the actual API response. It also contradicts the PR's own stated goal ("Remove useless RRM sort_order").
**Suggested fix:** Drop the RRM fallback: `'sort_order' => self::SA_SORT_ORDER[$id] ?? null`.
**Comment for the dev:** Since `allowedRmStatusIds()` and `SA_SORT_ORDER` cover the exact same id set, the `?? $status['SortOrder']` fallback here can never actually be returned to the FE (those statuses get filtered out). Given the PR description says we're removing the useless RRM `sort_order`, could we drop this fallback and go straight to `SA_SORT_ORDER[$id] ?? null`?

**File:** `app/Modules/Tasks/Services/TaskStatusService.php`
**Code snippet:**
```php
    public function list(): array
    {
        $result = $this->rentManagerService->getServiceManagerStatuses();
```
**What's wrong:** No feature test covers the new response shape/ordering of `GET /api/tasks/statuses`.
**Why:** `RULES.md` — "Every new or modified endpoint must have a feature test asserting the response shape; this is a hard requirement, not a suggestion." This PR changes both the `sort_order` values and the ordering of the list returned by this endpoint, but no test file is touched in the diff.
**Suggested fix:** Add a feature test asserting the response is sorted 0→7 by `sort_order` with the expected fixed values per status id.
**Comment for the dev:** Could you add a feature test for `GET /api/tasks/statuses` asserting the fixed `sort_order` values and the resulting order? Per team rules this is a hard requirement for any modified endpoint response shape.

### nit

**File:** `app/Modules/Tasks/Services/TaskStatusService.php`
**Code snippet:**
```php
        $filteredStatuses = array_values(array_filter(
            $mappedStatuses,
            fn (array $status) => in_array($status['id'], $allowedIds, true)
        ));

        usort($filteredStatuses, fn (array $a, array $b) => $a['sort_order'] <=> $b['sort_order']);
```
**What's wrong:** A runtime `usort` is used to reproduce an order that's already fully known and fixed at compile time (`SA_SORT_ORDER`).
**Why:** `AGENTS.md` already establishes this philosophy for the same kind of case: "`TaskStatus::cases()` returns enum cases in declaration order — no sort needed. Never add `usort` on top of it." Same idea applies here since `SA_SORT_ORDER` is a fixed, ordered map.
**Suggested fix:** Build the filtered/ordered list by iterating `SA_SORT_ORDER` (or `allowedRmStatusIds()`) and looking up each status by id, skipping any not present — no `usort` needed.
**Comment for the dev:** Minor: since `SA_SORT_ORDER` already fixes the order, you could build `$filteredStatuses` by iterating it directly instead of filter+usort — same idea as the "no sort needed for declaration-ordered data" rule we already have for `TaskStatus::cases()`. Not blocking.

## Verdict
needs changes before merge

## SPACE log
- Time spent reviewing: 30 min
- Model: Sonnet 5
- Findings: 0 blockers, 2 should-fix, 1 nit
- Verdict: needs changes before merge

## Review closed
- Timestamp: 2026-07-07
- Resolution: approved
- Notes: Dev pushed a fix addressing the findings; nothing left pending or waived.
- Time to read and validate the review: 30 min
