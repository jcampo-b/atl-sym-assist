## Review opened
- Timestamp: 2026-07-08
- Repo: FE
- Branch: feature/DEVSYM-278-cost-assignment-section
- Base branch: stage
- Issue: DEVSYM-278

## Dev's stated focus
Cost assignment section for task detail (Work orders / invoice cost tracking): add/edit/delete cost invoices with line items, bill-to selection, attachment upload, budget comparison, and editing lock once task status passes "Approved". Purely local state for now (no backend integration yet, tracked separately under DEVSYM-279).

## Files in scope
```
src/features/task-detail/TaskDetailPage.tsx
src/features/task-detail/components/ConfirmInvoiceModal.tsx
src/features/task-detail/components/CostInvoiceRow.tsx
src/features/task-detail/components/TaskDetailCosts.tsx
src/features/task-detail/components/TaskDetailMetaSidebar.tsx
src/features/task-detail/hooks/use-cost-management.ts
src/features/task-detail/types/cost.ts
src/features/task-detail/utils/cost.utils.test.ts
src/features/task-detail/utils/cost.utils.ts
src/stories/features/task-detail/TaskDetailCosts.stories.tsx
src/stories/features/task-detail/task-detail.stories.test.tsx
```

## Findings

### Should-fix

**File:** `src/features/task-detail/utils/cost.utils.ts`
**Code snippet:**
```
// ponytail: categoryId has no name mapping from the API yet (blocked on
// DEVSYM-276/279) — always show the section until that lands.
export function isMaintenanceTask(_task: TaskDetailScreenModel): boolean {
  return true
}
```
**What's wrong:** The function ignores its parameter and unconditionally returns `true`, making the `isMaintenanceTask(taskView) ? <TaskDetailCosts /> : null` branch in `TaskDetailPage.tsx` dead code that always takes the truthy path.
**Why:** AGENTS.md code quality rules on no speculative abstractions / no stub methods with no real logic.
**Suggested fix:** Drop `isMaintenanceTask` and render `<TaskDetailCosts />` unconditionally with a TODO(DEVSYM-276/279) comment, or keep the TODO inline at the call site instead of a wrapper function.

### Nit

**File:** `src/features/task-detail/utils/cost.utils.ts`
**Code snippet:**
```
// ponytail: no real property-level threshold data yet (approval workflows are
// out of scope per DESSYM-29, tracked separately in DESSYM-17) — static value
// matching the Figma mock until that data source exists.
export const MOCK_AUTO_APPROVAL_THRESHOLD = 150
```
**What's wrong:** Ticket references `DESSYM-29` / `DESSYM-17` look like typos for `DEVSYM-29` / `DEVSYM-17`.
**Why:** Wrong ticket key in a comment is unsearchable later.
**Suggested fix:** Fix the prefix, or confirm these are valid distinct ticket keys.

## Verdict
ready to merge

## SPACE log
- Time spent reviewing: 30 min
- Model: Sonnet 5
- Findings: 0 blockers, 1 should-fix, 1 nit
- Verdict: ready to merge

## Review closed
- Timestamp: 2026-07-08
- Resolution: approved
- Notes: Approved as-is; the should-fix (`isMaintenanceTask` stub) and nit (typo in ticket refs) were not addressed, waived by Johnny.
- Time to read/validate review: 10 min
