# DEVSYM-319 — State machine review: over-engineering corrections

## What happened
DEVSYM-319 state machine implementation was reviewed and found over-engineered in two areas.

## Corrections required

### 1. Remove StatusSequenceProvider + EnumSequenceProvider
These two files exist only to sort enum cases by internalId(). TaskStateMachine can do this inline:
  $cases = TaskStatus::cases();
  usort($cases, fn ($a, $b) => $a->internalId() <=> $b->internalId());
Delete both files. Remove the interface binding from TasksServiceProvider. Remove constructor injection from TaskStateMachine.

### 2. Remove buildUpdateBodyFromTask() from RentManagerIssueStatusMapper
A status update only needs ServiceManagerIssueID + StatusID. Sending Title and Description alongside is unnecessary and potentially harmful if RM rewrites them.
Build the body inline in TaskService::updateStatus(), same pattern as assign():
  [
      'ServiceManagerIssueID' => (int) $id,
      'StatusID'              => RentManagerIssueStatusMapper::toRmStatusId($target),
  ]

## Rule to remember
Before creating an interface or abstract class, ask: is there more than one concrete implementation today, or a confirmed ticket that requires one? If no → don't create it. Build what's needed now.

## Rule to remember
Before adding a helper method to a mapper, ask: is this called from more than one place? If it assembles a one-off payload that only one service uses → build it inline in that service.
