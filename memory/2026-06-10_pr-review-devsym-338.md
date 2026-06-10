# PR Review Findings — DEVSYM-338

## What was wrong

### Finding 1 — resolveScheduledDate() is dead logic
`MapsTaskRequestToData` had a `resolveScheduledDate()` method whose only body
was an `is_string` guard + null coalesce. The `date_format` + `nullable`
validation rules already guarantee the value is a valid string or null by the
time `toData()` runs — the guard is redundant.

### Finding 2 — `?? ''` sends blank Title to RM on missing Title edge case
`TaskService::update()` used `$raw['Title'] ?? ''` when pre-fetching the issue
before writing to RM. If RM returns an issue with no `Title` key, this sends
`Title: ''` — the same payload that triggers "Issue cannot be blank." from RM.
With `?? null`, `array_filter` in the mapper drops the key and RM keeps its
existing value.

## Files touched
- `app/Modules/Tasks/Requests/Concerns/MapsTaskRequestToData.php` — remove `resolveScheduledDate()`
- `app/Modules/Tasks/Requests/StoreTaskRequest.php` — inline `scheduledDate: $v['scheduled_date'] ?? null`
- `app/Modules/Tasks/Requests/UpdateTaskRequest.php` — same
- `app/Modules/Tasks/Services/TaskService.php` line 135 — `?? ''` → `?? null`

## Decisions made
- Two separate commits, one per finding.
- Commit 1: `fix[DEVSYM-338]: Request - inline scheduledDate mapping, remove redundant resolver`
- Commit 2: `fix[DEVSYM-338]: Service - use null fallback for missing Title in update pre-fetch`

## Rule to apply going forward
Simple field mapping in Request classes stays inline in `toData()` as
`$v['field'] ?? null`. Extract a resolver method only when there is actual
logic beyond a null coalesce (e.g. type casting, array transformation, format
normalization). Guards that duplicate what validation already enforces are dead
code — do not add them.
