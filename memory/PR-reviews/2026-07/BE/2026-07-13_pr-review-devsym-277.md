## Review opened
- Timestamp: 2026-07-13
- Repo: BE
- Branch: feature/devsym-277-rm-implement-cost-service-to-rentmanager
- Base branch: dev
- Issue: DEVSYM-277

## Dev's stated focus
Implement a cost service that syncs CostLineItems to RentManager ServiceManagerIssueWorkOrder lines: async create/update/delete queued jobs, sync-state tracking columns on `cost_line_items`, and the RM WorkOrder write methods.

## Files in scope
```
.env.example
RESTClient/rm-work-order-costs.http
app/Modules/Integrations/IntegrationsServiceProvider.php
app/Modules/Integrations/RentManager/Services/RentManagerService.php
app/Modules/Integrations/RentManager/Services/RentManagerWorkOrderService.php
app/Modules/OperativeCost/Enums/SyncStatus.php
app/Modules/OperativeCost/Jobs/DeleteWorkOrderLineFromRentManager.php
app/Modules/OperativeCost/Jobs/SyncCostLineItemToRentManager.php
app/Modules/OperativeCost/Models/CostLineItem.php
app/Modules/OperativeCost/Services/CostInvoiceService.php
app/Modules/OperativeCost/Services/CostLineItemService.php
config/rent_manager.php
database/migrations/2026_07_13_170000_add_rm_sync_columns_to_cost_line_items_table.php
tests/Feature/Modules/OperativeCost/CostInvoiceEndpointTest.php
tests/Feature/Modules/OperativeCost/CostLineItemEndpointTest.php
tests/Unit/Modules/Integrations/RentManager/Services/RentManagerWorkOrderServiceTest.php
tests/Unit/Modules/OperativeCost/Jobs/DeleteWorkOrderLineFromRentManagerTest.php
tests/Unit/Modules/OperativeCost/Jobs/SyncCostLineItemToRentManagerTest.php
tests/Unit/Modules/OperativeCost/Services/CostInvoiceServiceTest.php
tests/Unit/Modules/OperativeCost/Services/CostLineItemServiceTest.php
```

## Non-finding (verified, deliberately NOT flagged)
The SA-layer Job (`SyncCostLineItemToRentManager`) builds a PascalCase RM payload
(`ServiceManagerIssueID`, `InventoryItemID`, `Quantity`, `Price`, `Cost`, `Description`)
inline. This looks like a cardinal layer violation, but verified against
`app/Modules/Tasks/Services/TaskService.php` (lines ~172, 220-223, 255-258, 281-282):
the repo's de-facto convention for RM **writes** is to assemble the PascalCase payload
inline in the SA service and use mappers only for value translation
(`RentManagerIssueStatusMapper::toRmStatusId`). This PR is consistent with that. Not a finding.

Also verified: `Cost => null` is stripped by `PerformsRentManagerRequests::sanitizePayload()`
(recursive null removal) — consistent with the intended "leave RM Cost unset". Not a bug.
`RentManagerHttpClient::delete/post/get` and `RentManagerService::asSystem` all exist.
`CostInvoiceService` already injects `CostLineItemService` on `dev`.

## Findings

### should-fix
1. **`SyncCostLineItemToRentManager` create path is not idempotent under retry.**
   RM id is persisted only after `handle()` completes; if the RM POST succeeds but the
   job later throws (5xx post-write, transient error, or `normalizeWorkOrderRecord` can't
   find the id in an unseen shape), the retry runs `create` again with `$rmId` still null →
   duplicate RM WorkOrder line (up to `tries=5`). Update path is safe (carries the id).
   Fix: guard create against an existing line, OR document as a known at-least-once limitation.
   Snippet: `$rmId = $lineItem->service_manager_issue_work_order_id; $result = $rmId !== null ? update... : create...` + `failed()` sets `SyncStatus::Failed`.

### nit
2. **`SyncCostLineItemToRentManager` — "Re-check existence" comment is misleading.**
   `CostLineItem::whereKey($lineItem->id)->update([...])` doesn't re-check; it no-ops on 0 rows.
   Consequence: a create that completes after a local delete (whose `deleteLineItem` captured a
   still-null id and dispatched no cleanup) leaves an orphaned RM line. Reword + note the orphan gap.

3. **`CostInvoiceService::delete` no longer atomic.** Was a single statement; now N `deleteLineItem`
   calls + `$invoice->delete()` with no `DB::transaction` wrapper (unlike `create`). Wrap it — also
   gives the `afterCommit()` dispatches a real commit boundary.

4. **`.env.example` `RM_DEFAULT_INVENTORY_ITEM_ID=1` has no inline comment.** Magic id; add a one-line
   note (rationale lives in `config/rent_manager.php`). Aligned with the RULES.md `.env.example` convention.

## Verdict
needs changes before merge (no blockers; only #1 has real weight — if documented as a known
limitation instead of fixed, ready to merge)

## SPACE log
- Time spent reviewing: TBD
- Model: Opus 4.8
- Findings: 0 blockers, 1 should-fix, 3 nits
- Verdict: needs changes before merge
