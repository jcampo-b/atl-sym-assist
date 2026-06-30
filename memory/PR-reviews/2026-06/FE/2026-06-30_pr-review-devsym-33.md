## Review opened
- Timestamp: 2026-06-30
- Repo: FE
- Branch: DEVSYM-33-build-workflow-view-ex-kanban-frontend
- Base branch: stage
- Issue: DEVSYM-33

## Dev's stated focus
New Workflow page with status lanes, filters (status / property / unit / assignee), task detail popover, and sidebar nav entry.

## Files in scope
```
src/api/task/queries/list.queries.ts
src/api/task/task-list.api.ts
src/components/layout/AppSidebar.tsx
src/features/workflow/WorkflowPage.tsx
src/features/workflow/components/StatusCategoryRow.tsx
src/features/workflow/components/TaskDetailPopover.tsx
src/features/workflow/components/WorkflowFilters.tsx
src/features/workflow/components/WorkflowStatusDot.tsx
src/features/workflow/components/WorkflowTaskRow.tsx
src/features/workflow/components/WorkflowToolbar.tsx
src/features/workflow/hooks/use-workflow-filters.ts
src/routeTree.gen.ts
src/routes.config.ts
src/routes/_authenticated/workflow/index.tsx
```

## Findings

### should-fix

---

**File:** `src/features/workflow/components/TaskDetailPopover.tsx`
**Code snippet:**
```tsx
  metaParts.push(
    <MetaItem icon={<CalendarDays className="size-3.5" />}>{statusLabel}</MetaItem>,
  )
```
**Severity:** `should-fix`
**What's wrong:** `CalendarDays` (a calendar icon) is used for the status label — a text like "In Progress" or "Completed". This is semantically wrong and misleads the user into thinking it's a date field.
**Why:** The icon must semantically match the content it decorates. `CalendarDays` is used correctly one line above for a due date; reusing it here on a status name creates confusion.
**Suggested fix:** Use a neutral status icon (e.g., `CircleDot`) or omit the icon entirely for the status meta item.
**Comment for the dev:**
> The `CalendarDays` icon on the status label (line 85-86) is misleading — it's the same calendar icon used for dates, but here it decorates the status name ("In Progress", etc.). Consider a circle icon or remove the icon for this meta item.

---

**File:** `src/api/task/task-list.api.ts` + `src/features/workflow/hooks/use-workflow-filters.ts`
**Code snippet (api):**
```ts
  /** Wire: `property_id`. Backend support pending — forwarded for the workflow view filters. */
  propertyId?: number | null
  /** Wire: `unit_id`. Backend support pending — forwarded for the workflow view filters. */
  unitId?: number | null
```
**Code snippet (hook):**
```ts
  const listParamsForStatus = (categoryStatusId: number): ListTasksParams => ({
    statusId: categoryStatusId,
    q: debouncedQuery || undefined,
    assigneeId,
    propertyId: property?.propertyId ?? null,
    unitId: unit?.unitId ?? null,
  })
```
**Severity:** `should-fix`
**What's wrong:** Property and Unit filters are fully interactive in the UI but the backend ignores them. A user selecting "Property: Building A" will still see all tasks — the filter silently has no effect.
**Why:** Shipping interactive UI controls with no backend effect is a UX bug. The user has no way to know the filter is non-functional.
**Suggested fix:** Disable the Property and Unit filter pills (`disabled` prop + `opacity-50 cursor-not-allowed`) with a tooltip "Coming soon" until the backend ticket lands, or gate behind a feature flag.
**Comment for the dev:**
> Property and Unit filters are interactive but the backend doesn't process those params yet. Users will apply these filters and get unexpected results. Before merging, either disable the filter pills visually until the BE ticket lands, or add a tooltip/badge to indicate they're not active yet.

---

### nit

---

**File:** `src/features/workflow/hooks/use-workflow-filters.ts`
**Code snippet:**
```ts
  const hasActiveFilters =
    statusId != null ||
    assignee != null ||
    property != null ||
    unit != null ||
    debouncedQuery.length > 0

  const clearAll = () => {
    setQuery('')
    setStatusId(null)
    setAssignee(null)
    setProperty(null)
    setUnit(null)
  }
```
**Severity:** `nit`
**What's wrong:** `hasActiveFilters` and `clearAll` are returned by the hook but never consumed anywhere. Dead return surface.
**Why:** No speculative abstractions — code-review skill. If there's no "Clear all" button yet, don't export the machinery for it.
**Suggested fix:** Remove both from the hook's return object. Add them back in the PR that implements the "Clear all" button.
**Comment for the dev:**
> `hasActiveFilters` and `clearAll` are returned by `useWorkflowFilters` but nothing reads them. Remove them for now and add them back when the "Clear all" UI lands.

---

## Verdict
needs changes before merge

## SPACE log
- Time spent reviewing: 20 min
- Model: claude-sonnet-4-6
- Findings: 0 blockers, 2 should-fix, 1 nit
- Verdict: needs changes before merge

## Review closed
- Timestamp: 2026-06-30
- Resolution: approved
- Notes: PR approved as-is. 2 should-fix findings (wrong icon + pending filters) and 1 nit acknowledged but waived by Johnny for this merge.
