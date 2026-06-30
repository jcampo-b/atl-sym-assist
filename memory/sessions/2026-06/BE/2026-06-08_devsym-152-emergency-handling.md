# DEVSYM-152 — Emergency handling logic

## What was done
Implemented emergency bypass path for PATCH /api/tasks/{id}/status.
Emergency tasks skip assertCanAdvance(), go directly to Dispatch, and write
an audit note to the RM activity log via createIssueNote().

## Files touched

### Created
- app/Modules/Tasks/StateMachine/EmergencyContext.php — readonly value object (justification, flaggedByEmail, flaggedById)
- tests/Unit/Modules/Tasks/StateMachine/EmergencyContextTest.php
- tests/Feature/Modules/Tasks/EmergencyStatusBypassTest.php

### Modified
- app/Modules/Tasks/Requests/UpdateTaskStatusRequest.php — added is_emergency / emergency_justification validation, toEmergencyContext()
- app/Modules/Tasks/Controllers/TaskController.php — passes toEmergencyContext() as third arg to taskService->updateStatus()
- app/Modules/Tasks/Services/TaskService.php — emergency path in updateStatus(): skips assertCanAdvance(), enforces Dispatch target, writes RM status, writes audit note, logs

## Decisions made
- Emergency context is resolved inside UpdateTaskStatusRequest::toEmergencyContext() using $this->user('sanctum'), not in the controller. This keeps the controller a one-liner.
- Audit trail is a RM History Note written via createIssueNote() — no new DB table needed.
- Note format: "EMERGENCY bypass — flagged by {email} (id: {id}). Justification: {justification}. Dispatched at: {iso8601}."
- Notification in this ticket = activity log only. Email/SMS/push deferred to separate notifications ticket.
- Rule engine does NOT run on emergency transitions — it fires after assertCanAdvance(), which is skipped.
- toEmergencyContext() guards against null user with RuntimeException before accessing email/id.

## Open questions
- RM NoteType for createIssueNote() — currently passing no explicit type. Confirm if a specific NoteType should be used for audit notes.
