# PR Review Corrections — DEVSYM-319

## What was wrong
Five issues found during PR review by jona872.

## Corrections

### 1. Title missing from auto-transition RM update
TransitionRuleEngine::applyRule() was missing Title in the updateServiceManagerIssue
body — same bug that was fixed in TaskService::updateStatus(). Every auto-transition
was silently failing with "Issue cannot be blank." from RM.
Fix: add 'Title' => $task['title'] ?? '' to the body in applyRule().

### 2. Redundant usort() in TaskStateMachine::next()
TaskStatus::cases() already returns enum cases in declaration order (1–7).
The usort() was resorting something already sorted on every call.
Fix: remove usort(), iterate cases() directly.

### 3. ExceptionContext.php — premature stub
Empty DTO with no fields, no tests, no callers. Belongs in DEVSYM-152 when
the required fields are known.
Fix: delete the file, remove its import from TaskStateMachine.php.

### 4. applyException() — premature stub
Method does nothing, has no tests, is not called from anywhere.
Fix: remove the method. Update the class docblock to document the intent:
"Out-of-sequence transitions (emergency bypass, rework) will be implemented
in DEVSYM-152 via a dedicated method that does not go through assertCanAdvance()."

### 5. 7 stub rule files — premature
BudgetUnderThresholdRule, EmergencyFlagRule, InvoicePaymentRule,
PreferredVendorAvailableRule, ReviewApprovedRule, VendorCompletionRule,
VendorEvidenceRule — all return false with no logic.
Each belongs in its own ticket. EmergencyFlagRule specifically does not belong
in the rule engine at all — emergency bypass happens before assertCanAdvance(),
not after.
Fix: delete all 7 files. Clean up TasksServiceProvider to register only
AllSubtasksCompleteRule.

## Rules to apply going forward
- Never ship stub files or no-op methods as placeholders. Create them in the
  ticket where the logic is implemented.
- TaskStatus::cases() returns cases in declaration order — never usort on top of it.
- The rule engine fires AFTER a transition is written. Logic that bypasses
  assertCanAdvance() (emergency, rework) goes in TaskService::updateStatus(),
  not in a rule.
