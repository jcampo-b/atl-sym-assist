# Cleanup Note — resolveDueDate() redundant is_string guard

## What
`resolveDueDate()` in `MapsTaskRequestToData` has the same redundant `is_string`
guard pattern that was removed from `resolveScheduledDate()` in DEVSYM-338.

## Do not touch now
Scope of DEVSYM-338 is limited to the two PR review findings. Address this in
a dedicated cleanup ticket.

## Rule
`date` / `date_format` + `nullable` validation already guarantees the value is
a valid string or null — the guard is dead code by the same logic.

## File
`app/Modules/Tasks/Requests/Concerns/MapsTaskRequestToData.php` — `resolveDueDate()`
