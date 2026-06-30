# PR Review Corrections — DEVSYM-152

## What was wrong

### 1. Emergency status constraint in the wrong layer
Service threw InvalidTransitionException when is_emergency=true and status != dispatch.
That is input validation — it belongs in UpdateTaskStatusRequest, not the service.
Rule: input constraints go in the Request layer. Business logic goes in the Service.

### 2. Emergency transition validation bypasses TaskStateMachine
Service had a ~30-line inline block that bypassed the state machine entirely.
Emergency transition rules belong in the state machine.
Rule: TaskStateMachine is the single source of truth for all valid transitions,
including emergency ones.

### 3. Dead defensive code in wrong layer
RuntimeException guard for null user in toEmergencyContext() — auth:sanctum 
middleware guarantees $this->user() is never null on authenticated routes.
Rule: never guard against conditions that middleware already prevents.

### 4. Non-idiomatic boolean check
empty($v['is_emergency']) on the validated array instead of $this->boolean('is_emergency').
Rule: always use FormRequest helpers (boolean(), integer(), string()) over
raw array access on $this->validated().

### 5. Unnecessary RM round-trip
$this->get($id) at the end of the emergency path fires an extra RM API call.
updateServiceManagerIssue() already returns the updated data — use it directly.
Rule: never re-fetch from RM just to format a response unless strictly necessary.

## Rules to apply going forward
- Input validation (field constraints, allowed values) belongs in FormRequest rules().
- TaskStateMachine owns all transition logic including emergency path.
- Never guard null on $this->user() on sanctum-authenticated routes — middleware guarantees it.
- Use FormRequest helpers: boolean(), integer(), string() — never empty($v['field']).
- Use the return value of updateServiceManagerIssue() directly; don't re-fetch.
