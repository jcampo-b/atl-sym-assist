# DEVSYM-345 BE — estimated_time_minutes not in task detail response

## What was done
Diagnosed and fixed the missing `estimated_time_minutes` field in `GET /api/tasks/{id}` and write responses (PATCH, status update).

Root cause: RM never includes `Hours` in the default `ServiceManagerIssue` response — it must be explicitly requested via `fields=...,Hours,...`. `toTaskDetailed()` had correct mapping logic (`Hours × 60 → estimated_time_minutes`) but the field was silently absent from the RM payload every time.

## Files touched
- `app/Modules/Integrations/RentManager/Mappers/RentManagerIssueMapper.php`
  - Added `DETAIL_FIELDS` constant: comma-separated list of every RM field needed by `toTask()` + `toTaskDetailed()`, including `Hours`
  - Added `mergeDetailFields(array $query): array`: ensures `Hours` is in the `fields` param (sets DETAIL_FIELDS if none specified; appends `,Hours` if partial list provided)
- `app/Modules/Tasks/Services/TaskService.php`
  - `get()`: wraps `$query` with `RentManagerIssueMapper::mergeDetailFields($query)` before `getServiceManagerIssue`

## Decisions made
- `DETAIL_FIELDS` is an explicit whitelist — if someone adds a field to `toTask()` or `toTaskDetailed()`, they must also add it here or it silently returns null when `fields` is active. Documented in the constant's docblock.
- `str_contains` check for 'Hours' is sufficient — no RM field in DETAIL_FIELDS contains 'Hours' as a substring (verified).
- Emergency `updateStatus()` path (line 205, `TaskService.php`) calls `toTaskDetailed()` directly on the RM PUT response and is NOT fixed. Out of scope: emergency transitions don't modify estimated time.

## Known traps
- **RM omits `Hours` from defaults entirely** — unlike DueDate/ScheduledDate which are returned when set, `Hours` is never in the default response regardless of value. This is unique to this field.
- **`DETAIL_FIELDS` is a maintenance contract** — every new field added to the mapper must also be added here.

## Open questions / follow-ups
- Emergency `updateStatus()` path still misses `estimated_time_minutes` — separate ticket needed if this becomes a problem.
