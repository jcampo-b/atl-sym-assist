## Review opened
- Timestamp: 2026-07-14
- Repo: BE
- Branch: feature/devsym-277-rm-implement-cost-service-to-rentmanager
- Base branch: dev
- Issue: DEVSYM-277

## Dev's stated focus
No explicit focus provided with the review request. Inferred from the branch
name: implement the RM cost service — sync CostLineItem CRUD (create/update/
delete) to Rent Manager `ServiceManagerIssueWorkOrder` lines via queued jobs,
persisting the RM id + a sync_status on each line item.

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

## Findings

### Blockers

**File:** `app/Modules/OperativeCost/Jobs/SyncCostLineItemToRentManager.php`
Code snippet:
```php
$payload = [
    'ServiceManagerIssueID' => (int) $lineItem->costInvoice->task_id,
    'InventoryItemID' => config('rent_manager.default_inventory_item_id'),
    'Quantity' => (float) $lineItem->qty,
    'Price' => (float) $lineItem->sale,
    // SA only captures the billed price, never an internal cost —
    // leave RM's Cost unset rather than fabricating a figure SA
    // doesn't have (locked decision, DEVSYM-277).
    'Cost' => null,
    'Description' => $lineItem->description,
];
```
- Severity: blocker
- What's wrong: Full SA→RM field translation built inline with PascalCase RM
  field names inside an SA-layer class (`app/Modules/OperativeCost/`).
- Why: Cardinal layer rule (AGENTS.md — PascalCase outside
  `app/Modules/Integrations/RentManager/` = STOP) + RULES.md "Layer discipline"
  (layer discipline wins even for a single-caller method). Precedent:
  `TaskService` delegates its write payload to `RentManagerIssueMapper::fromTaskData()`
  and only spreads the raw RM id inline; DEVSYM-277 has no mapper call at all.
- Suggested fix: `RentManagerWorkOrderMapper` already exists (read-only:
  `toWorkOrder()`). Add a write mapper there — e.g. `fromCostLineItem(CostLineItem): array`
  — owning the PascalCase keys, the `InventoryItemID` default, and the
  `Cost => null` decision. Job delegates to it.
- Comment for dev: (see chat — ready to paste)
- **RESOLVED in commit dd0620a (re-reviewed 2026-07-14):** dev added
  `RentManagerWorkOrderMapper::fromCostLineItem()` in the RM layer and the job
  now calls it (`$payload = RentManagerWorkOrderMapper::fromCostLineItem($lineItem)`).
  SA layer no longer builds the PascalCase business-field payload. The
  remaining `ServiceManagerIssueWorkOrderID` merge/read-back in the job is a
  raw RM-identifier passthrough at the call site — accepted precedent
  (mirrors TaskService spreading `ServiceManagerIssueID`/`Title`), not a layer
  violation. Added `RentManagerWorkOrderMapperTest` covering the mapping +
  null-Cost decision. Blocker cleared.

### Nits

**File:** `app/Modules/OperativeCost/Jobs/SyncCostLineItemToRentManager.php`
Code snippet:
```php
if ($lineItem === null) {
    return;
}
```
- Severity: nit
- What's wrong: Residual create-then-delete orphan window — if the local row is
  deleted after this job's RM create call but before the write-back update(),
  the delete path saw a null RM id and dispatched no cleanup → orphaned
  WorkOrder line in RM.
- Why: RM is system of record; silent orphans are hard to detect. Low
  probability (sub-second race, concurrent create+delete on a new row).
- Suggested fix: No code change required — document the window in a comment, or
  a reconciliation follow-up ticket if orphans ever matter operationally.

**File:** `.env.example`
Code snippet:
```
RM_DEFAULT_INVENTORY_ITEM_ID=1
```
- Severity: nit
- What's wrong: No inline comment; bare `1` reads as a meaningful id when it's a
  fixed placeholder ("Maintenance and Repair Charge", Product decision).
- Why: RULES.md — .env.example is often the first setup doc a dev reads;
  annotate non-obvious values. config/rent_manager.php documents it well,
  .env.example gives no hint.
- Suggested fix: One-line comment above the var.

**File:** `app/Modules/Integrations/RentManager/Services/RentManagerWorkOrderService.php`
Code snippet:
```php
if (! array_key_exists('ServiceManagerIssueWorkOrderID', $raw) && isset($raw['data']) && is_array($raw['data'])) {
    return $this->normalizeWorkOrderRecord($raw['data']);
}

if (isset($raw['items']) && is_array($raw['items']) && $raw['items'] !== []) {
    $items = $raw['items'];
    if (array_is_list($items) && isset($items[0]) && is_array($items[0])) {
        return $this->normalizeWorkOrderRecord($items[0]);
    }
}
```
- Severity: nit
- What's wrong: Only list-wrap and flat-object shapes are confirmed-live and
  unit-tested. The `data`-key / `items`-key branches handle shapes not observed
  for this endpoint and have no test.
- Why: AGENTS.md "No speculative abstractions." Comment justifies parity with
  `RentManagerIssueService::normalizeIssueRecord()` — reasonable if intentional.
- Suggested fix: Confirm parity is deliberate (leave) or drop the two
  unconfirmed/untested branches.

## Positives (not findings)
- `->afterCommit()` on all dispatches with a clear comment on why.
- Honest RM contract comments (destructive full-replace upsert, null Cost,
  response-shape ambiguity), cross-referenced to a write-contract doc.
- Good test coverage: RM service (payload/normalize/422), both jobs
  (create/update/no-op/failed), both SA services (dispatch assertions via
  Queue::fake), feature tests faked the queue.
- Migration correctly avoids FK for the RM id and uses a new timestamped file.

## Verdict
- Initial: needs changes before merge (1 blocker — layer discipline; 3 nits optional)
- After fix (dd0620a, 2026-07-14): ready to merge. Blocker resolved. 3 nits
  remain unaddressed but were optional.

## SPACE log
- Time spent reviewing: 30 min
- Model: Opus 4.8
- Findings: 1 blocker, 0 should-fix, 3 nits
- Verdict: needs changes before merge
