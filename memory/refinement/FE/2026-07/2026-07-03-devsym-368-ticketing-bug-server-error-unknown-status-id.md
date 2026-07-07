# DEVSYM-368 — [Ticketing] Bug: "Server Error" al intentar cambiar status en tareas con status ID desconocido
> Refined 2026-07-03. Source of truth for implementation is still the Linear ticket + AGENTS.md.

## Current ask (as read from Linear)
Tasks whose current RM status ID is not in the FE `RM_STATUS_ID_TO_ENUM` map (e.g. "Status 7", "Status 4", "Status 19") show a "Server Error" when the user tries to change status from the detail screen, because `toStatusEnum()` throws on an unmapped ID. A documented special case: selecting `<Unassigned>` (RM status ID 1) shows "Unknown RM status ID: 1". Expected: the status selector should be disabled or show a clear message when the current status is unrecognized, instead of surfacing a raw error.

## Target
Repo(s): FE  •  Module: task-detail / task-search status selector  •  Layer(s): feature component + api mutation  •  RM-backed: no (pure FE; no BE/RM change)
Contract source: none (FE-only). Relevant files: `update-task-status.mutations.ts`, `TaskStatusSelect.tsx`, `TaskDetailMetaSidebar.tsx`, `statuses.queries.ts`.

## Refined scope
Root cause: the FE has TWO independent sources of truth for statuses that don't agree.
1. The selectable target options come from `GET /tasks/statuses` (full active RM status list) via `mapTaskStatusesToSelectOptions` — this list is broader than the writable set and can include RM status ID 1 (`<Unassigned>`), "Status 4/7/19", etc.
2. The write path (`toStatusEnum` in `update-task-status.mutations.ts`) only accepts the 7 IDs in `RM_STATUS_ID_TO_ENUM` (56/52/53/54/55/18/22) and `throw`s a raw `Error` for anything else. That thrown message is surfaced to the user via `getApiErrorMessage`/toast.

IN scope: make the detail-screen status selector never call `toStatusEnum` with a non-writable ID. Concretely: restrict the selectable *target* options to the writable set (`RM_STATUS_ID_TO_ENUM`), and render an unknown *current* status non-destructively (the "orphan" option in `TaskStatusSelect.tsx:27-34` already displays it as `Status <id>`; make it non-selectable / disable the change affordance so no throw is possible). Extend the existing test (`update-task-status.mutations.test.ts`) to cover the guard.

NOT in scope: any BE change to `GET /tasks/statuses` (it legitimately returns the full list for search/filter use); adding a refresh/write endpoint; changing the BE status lifecycle mapping. Do not turn the writable-status list into a new abstraction/config — the existing `RM_STATUS_ID_TO_ENUM` constant is the FE gate.

## Comment ledger
No comments on the ticket. Nothing to incorporate, exclude, or escalate from the thread.

## Contradictions / open questions
No direct section-vs-section contradiction (the two phrases below are on different axes — one is a *target*-selection failure, the other a *current*-status remedy). But their combination leaves one genuine scope gap the ticket does not resolve:

Context (special case): "Un caso especial: intentar cambiar al status `<Unassigned>` muestra "Unknown RM status ID: 1""
Expected behavior:      "El selector de status debería estar deshabilitado o mostrar un mensaje claro cuando el status actual no es reconocido"
→ **→ Lean** — The prescribed remedy only covers the case where the *current* status is unrecognized. The documented `<Unassigned>`/ID-1 failure is a *target*-selection failure that also occurs when the current status IS recognized (a normal task with the Unassigned option offered). Does the fix need to also stop non-writable statuses (ID 1 and any status not in `RM_STATUS_ID_TO_ENUM`) from being *selectable targets*, or is handling only the unrecognized-current-status display considered done? Recommended reading: cover both (filter target options to the writable set); confirm with Lean before closing.

## Architecture decisions that apply
- FE consumes only SA snake_case contracts; this fix stays entirely on the FE display/write path — no RM field names, no BE contract change (AGENTS.md layer rule).
- Code-quality: no speculative abstraction — reuse the existing `RM_STATUS_ID_TO_ENUM` constant as the writable gate; do not introduce a config/interface for the writable set (AGENTS.md "no speculative abstractions").
- Every modified FE behavior with a testable unit gets a test — extend the existing `update-task-status.mutations.test.ts` rather than adding a parallel one.

## Traps in play
- **Dual Virgin IDs (RM read vs write).** RM assigns `ServiceManagerStatusID: 1` (`<Unassigned>`) to newly-created issues on the READ path; the writable Virgin ID is 56. `RM_STATUS_ID_TO_ENUM` only contains 56, so ID 1 is correctly non-writable — the fix must treat ID 1 as a non-selectable target, not add it to the map. (AGENTS.md known traps + skill preamble "dual Virgin IDs".)
- Canonical writable status IDs: Virgin=56, Approval=52, Dispatch=53, Production=54, Review=55, Billing=18, Completed=22 — matches the current `RM_STATUS_ID_TO_ENUM`. Do not add ID 1.

Not in play (checked): the `invalidateQueries` vs `refetchQueries` FE trap — the mutation's `onSuccess` uses `invalidateAllTaskQueries`, but this ticket is about preventing a throw, not refresh timing; leave it. The `@/api/dashboard/keys` stage-only trap — this change does not touch dashboard keys.

## Drift / flags
None. FE manifest (§7 grep, verbatim): `@tanstack/react-query: "^5.90.21"`, `react: "^19.2.0"`, `zod: "^4.3.6"`. The ticket makes no version assumption to contradict. ⚠️ Open questions flag applies (see Contradictions / open questions).

## Housekeeping (optional)
Non-blocking observation (not a drift): `TaskStatusSelect.tsx:27-34` already adds an "orphan" option so an unknown *current* status renders as `Status <id>` without crashing on load. This means the ticket's original symptom ("Server Error on tasks with unknown current status", filed 2026-06-19) may have partially shifted — the remaining crash is at write-time when a non-writable target is selected. The dev should verify current live behavior before assuming the render-path crash still reproduces.

## Suggested estimate
2 points — single FE feature area, ~1–2 files (`TaskStatusSelect`/mutation guard), an existing unit test to extend, no BE/RM round-trips. The one open question (target filtering scope) does not change the size materially.
