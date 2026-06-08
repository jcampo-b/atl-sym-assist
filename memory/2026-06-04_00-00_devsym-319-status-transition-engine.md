# DEVSYM-319 — Status transition automation engine

## What was done
Implemented the full lifecycle state machine and auto-transition rule engine for ServiceManager tasks.

## Files touched

### Created
- `app/Modules/Tasks/StateMachine/TaskStatus.php` — backed string enum, internalId() 1–7
- `app/Modules/Tasks/StateMachine/StatusSequenceProvider.php` — interface
- `app/Modules/Tasks/StateMachine/EnumSequenceProvider.php` — MVP impl
- `app/Modules/Tasks/StateMachine/ExceptionContext.php` — empty seam for DEVSYM-152/320
- `app/Modules/Tasks/StateMachine/TaskStateMachine.php` — next(), canAdvanceTo(), assertCanAdvance(), applyException() seam
- `app/Modules/Tasks/StateMachine/Exceptions/InvalidTransitionException.php` — 422 via render()
- `app/Modules/Tasks/StateMachine/Rules/TransitionRule.php` — interface
- `app/Modules/Tasks/StateMachine/Rules/TransitionRuleEngine.php` — first-match wins, catches own errors
- `app/Modules/Tasks/StateMachine/Rules/AllSubtasksCompleteRule.php` — REAL: production→review when all subtasks closed
- `app/Modules/Tasks/StateMachine/Rules/BudgetUnderThresholdRule.php` — stub, TODO DEVSYM-149
- `app/Modules/Tasks/StateMachine/Rules/EmergencyFlagRule.php` — stub, TODO DEVSYM-152
- `app/Modules/Tasks/StateMachine/Rules/PreferredVendorAvailableRule.php` — stub
- `app/Modules/Tasks/StateMachine/Rules/VendorCompletionRule.php` — stub
- `app/Modules/Tasks/StateMachine/Rules/VendorEvidenceRule.php` — stub
- `app/Modules/Tasks/StateMachine/Rules/ReviewApprovedRule.php` — stub
- `app/Modules/Tasks/StateMachine/Rules/InvoicePaymentRule.php` — stub
- `tests/Unit/Modules/Tasks/StateMachine/TaskStateMachineTest.php`
- `tests/Feature/Modules/Tasks/TaskStatusTransitionTest.php`

### Modified
- `app/Modules/Integrations/RentManager/Mappers/RentManagerIssueStatusMapper.php` — added toRmStatusId(), fromRmStatusId(), resolveFromRawIssue(), buildUpdateBodyFromTask(), allowedRmStatusIds(); updated STATUS_STYLES with confirmed IDs
- `app/Modules/Tasks/Services/TaskService.php` — updateStatus() now takes TaskStatus, routes through machine + engine
- `app/Modules/Tasks/Services/TaskStatusService.php` — replaced hardcoded ALLOWED_STATUS_IDS with mapper call
- `app/Modules/Tasks/Requests/UpdateTaskStatusRequest.php` — status (string) replaces status_id (int); BREAKING CHANGE
- `app/Modules/Tasks/Controllers/TaskController.php` — updateStatus() calls toTaskStatus()
- `app/Modules/Tasks/TasksServiceProvider.php` — binds StatusSequenceProvider + TransitionRuleEngine
- `RESTClient/tasks.http` — updated PATCH status examples

## Decisions made

### RM ID ↔ lifecycle mapping (confirmed by dev lead 2026-06-04)
| SA value    | RM ServiceManagerStatusID | RM Name   |
|-------------|---------------------------|-----------|
| virgin      | 56                        | Virgin    |
| approval    | 52                        | Approval  |
| dispatch    | 53                        | Dispatch  |
| production  | 54                        | Production|
| review      | 55                        | Review    |
| billing     | 18                        | BILLING   |
| completed   | 22                        | Completed |

### API breaking change (confirmed intentional)
`PATCH /api/tasks/{id}/status` body changed from `{ "status_id": 52 }` to `{ "status": "approval" }`.

### Layer architecture upheld
- All RM PascalCase field names remain inside `app/Modules/Integrations/RentManager/Mappers/RentManagerIssueStatusMapper.php`
- TaskStatus enum never holds or references RM IDs
- TransitionRuleEngine depends on SA (TaskStateMachine) and RM (RentManagerService, mapper) — correct direction

### Engine error handling
Engine catches all errors (InvalidTransitionException + Throwable) and logs them without rethrowing. The HTTP response is never affected by auto-transition failures. Manual-review flagging deferred until a dedicated mechanism exists.

### Feature tests need RefreshDatabase
The test environment uses SQLite in-memory. All feature tests need `use RefreshDatabase` for `actingAs()` to work — the users table must exist. Existing feature tests in this project do NOT have this trait and are all failing with 401. This is a pre-existing environment issue unrelated to this ticket.

## Open questions
- Activity-log writes (RM History Notes) for transitions are currently done via Log::info() only. Confirm whether RM notes should be written via createIssueNote() and what NoteType to use.
- Manual-review flagging mechanism — no dedicated feature exists yet.
- Rule ordering: AllSubtasksCompleteRule is first. Confirm desired evaluation order for production rules when implemented.
