# Known Trap: RM assigns ServiceManagerStatusID 1 (<Unassigned>) to new issues

## What was done
Added ID 1 → TaskStatus::Virgin to `RM_TO_LIFECYCLE` in `RentManagerIssueStatusMapper`.

## Files touched
- `SymAssist-Backend/app/Modules/Integrations/RentManager/Mappers/RentManagerIssueStatusMapper.php`

## What was wrong
When a task is created in RM without an explicit status, RM assigns
`ServiceManagerStatusID: 1` (`<Unassigned>`). This ID was not in the lifecycle
mapping, causing a 500 on `PATCH /tasks/{id}/status` with:
"Unknown RM ServiceManagerStatusID: 1"

## Rule
ID 1 is read-only mapped to Virgin. It is intentionally absent from
`LIFECYCLE_TO_RM` — any status write uses ID 56 (the real Virgin). This is a
read-path alias only.

## Open questions
- Product may want to enforce ID 56 at creation time (option 2) to avoid the
  conceptual ambiguity of <Unassigned> vs Virgin. Tracked as a product decision.
