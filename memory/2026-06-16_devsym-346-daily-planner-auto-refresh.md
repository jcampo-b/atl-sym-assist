# DEVSYM-346 — Daily Planner auto-refresh on task delete

## What was done
Fixed the FE side of DEVSYM-346: after a task is deleted, the Daily Planner now
refetches immediately instead of requiring a manual page reload.

## Files touched
- `SymAssist-Frontend/src/api/task/mutations/delete-task.mutations.ts`
  — added `dashboardKeys` import, added `refetchQueries({ queryKey: dashboardKeys.all })` in `onSuccess`

## Decisions made

### `refetchQueries` over `invalidateQueries`
`invalidateQueries` was tried first but caused a ~2-minute delay before the planner
updated. Root cause: the global QueryClient (`src/lib/query-client.ts`) has
`staleTime: 1000 * 60` (1 min). With navigation timing (afterOptimisticRemoval
fires navigate() in onMutate, then onSuccess fires after API responds), `invalidateQueries`
deferred the actual network request until the stale window elapsed.
`refetchQueries` bypasses the stale check entirely — use it whenever you need
guaranteed immediate sync after a mutation.

### Key namespace
Daily Planner queries are under `dashboardKeys.all = ['dashboard']` — separate from
`taskKeys.all = ['tasks']`. `invalidateAllTaskQueries` never touched the planner.
Always check the key factory before assuming task mutations cover dashboard queries.

### Branch
`fix/devsym-346-daily-planner-auto-refresh` — branched from dev, 1 commit, pushed.
PR not yet opened.

## Open questions / follow-ups
- Other mutations (status change, reassignment) also don't refetch the planner.
  Flagged during Step 1 discovery — separate ticket.
- `dailyPlannerPendingInfiniteQueryOptions` has its own `staleTime: 30_000` override
  — consistent with the same `refetchQueries` fix being needed there too if those
  mutations are addressed.
