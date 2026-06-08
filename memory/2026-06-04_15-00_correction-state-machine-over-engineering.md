# Correction — DEVSYM-319 state machine over-engineering

## What was wrong
DEVSYM-319 state machine implementation was over-engineered in two areas.

## Corrections

### 1. StatusSequenceProvider + EnumSequenceProvider — removed
These two files existed only to sort enum cases by internalId(). TaskStateMachine
can do this inline:
  $cases = TaskStatus::cases();
  usort($cases, fn ($a, $b) => $a->internalId() <=> $b->internalId());
Both files deleted. Interface binding removed from TasksServiceProvider.
Constructor injection removed from TaskStateMachine.

### 2. buildUpdateBodyFromTask() — removed from RentManagerIssueStatusMapper
A status update only needs ServiceManagerIssueID + StatusID. Sending Title and
Description alongside is unnecessary and potentially harmful if RM rewrites them.
Body built inline in TaskService::updateStatus(), same pattern as assign():
  [
      'ServiceManagerIssueID' => (int) $id,
      'StatusID'              => RentManagerIssueStatusMapper::toRmStatusId($target),
  ]

## Rules to apply going forward
- Before creating an interface or abstract class: is there more than one concrete
  implementation today, or a confirmed ticket that requires one? If no → inline it.
- Before adding a method to a mapper: is it called from more than one place? If it
  builds a payload used by only one caller → build it inline at the call site.
