# Known Trap: RM rejects ServiceManagerIssue updates without Title

## What was done
Added `Title` to the `updateServiceManagerIssue` call in `TaskService::updateStatus()`.

## Files touched
- `SymAssist-Backend/app/Modules/Tasks/Services/TaskService.php` — line ~176

## Decisions made
- `$raw['Title'] ?? ''` — `$raw` is already fetched in step 1 of `updateStatus()`, so no extra RM call needed.
- Empty string fallback is safe: RM allows empty Title on update; it is not the same as omitting the field entirely.

## What was wrong
RentManager returns `"Issue cannot be blank."` when a PATCH to `ServiceManagerIssues` omits `Title`. The field is required by RM even when the caller only intends to change `StatusID`.

## Rule
**Always carry `Title` from the fetched raw issue in any RM update.**
Any call to `updateServiceManagerIssue` must include `Title` alongside whatever fields are being changed.

## Open questions
None.
