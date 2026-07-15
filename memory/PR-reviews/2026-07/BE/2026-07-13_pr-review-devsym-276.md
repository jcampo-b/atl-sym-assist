## Review opened
- Timestamp: 2026-07-13 10:46 -03
- Repo: BE
- Branch: feature/devsym-276-operative-costs-crud-api
- Base branch: dev
- Issue: DEVSYM-276

## Dev's stated focus
Operative costs CRUD API — new `OperativeCost` module: local `cost_invoices` + `cost_line_items`
tables scoped by RM `task_id` (string ref, no FK), CRUD endpoints for invoices and their line
items, totals (`totals_by_type` + `grand_total`) in the list response.

## Files in scope
RESTClient/cost-invoices.http
app/Modules/OperativeCost/Controllers/CostInvoiceController.php
app/Modules/OperativeCost/Controllers/CostLineItemController.php
app/Modules/OperativeCost/DTOs/CostInvoiceData.php
app/Modules/OperativeCost/DTOs/CostLineItemData.php
app/Modules/OperativeCost/Enums/BillToTarget.php
app/Modules/OperativeCost/Enums/ItemType.php
app/Modules/OperativeCost/Models/CostInvoice.php
app/Modules/OperativeCost/Models/CostLineItem.php
app/Modules/OperativeCost/OperativeCostServiceProvider.php
app/Modules/OperativeCost/Requests/StoreCostInvoiceRequest.php
app/Modules/OperativeCost/Requests/StoreCostLineItemRequest.php
app/Modules/OperativeCost/Requests/UpdateCostInvoiceRequest.php
app/Modules/OperativeCost/Requests/UpdateCostLineItemRequest.php
app/Modules/OperativeCost/Services/CostInvoiceService.php
app/Modules/OperativeCost/Services/CostLineItemService.php
app/Modules/OperativeCost/routes.php
bootstrap/providers.php
database/migrations/2026_07_10_140459_create_cost_invoices_and_line_items_tables.php
storage/api-docs/api-docs.json
tests/Feature/Modules/OperativeCost/CostInvoiceEndpointTest.php
tests/Feature/Modules/OperativeCost/CostLineItemEndpointTest.php
tests/Unit/Modules/OperativeCost/Services/CostInvoiceServiceTest.php
tests/Unit/Modules/OperativeCost/Services/CostLineItemServiceTest.php

## Verified, NOT a finding
- Route param naming: `apiResource('tasks.cost-invoices')` generates `{cost_invoice}` (snake)
  while the controller method uses `string $costInvoice` (camel). This is the identical pattern
  already shipped in `UserProfileController::update(..., string $userProfile)` with
  `apiResource('user-profiles')` → `{user_profile}`. Laravel resolves it; not a bug.
- Layer discipline OK: `CostInvoiceService` injects `RentManagerService` (the SA-facing
  integration service, same as `TaskService`), NOT `RentManagerHttpClient`. No RM PascalCase
  leaks into the SA layer. RM call is a pure existence check for the task (→ 404), outside the
  DB transaction. Correct.
- `ItemType`/`BillToTarget` enums: no `usort` on `::cases()`; declaration order relied upon for
  `totals_by_type`. Matches AGENTS.md rule.
- `list()` eager-loads `lineItems` (no N+1). Cross-task scoping enforced on every
  read/update/delete via `where('task_id', ...)`. Covered by tests.

## Findings

### SHOULD-FIX

**File:** `app/Modules/OperativeCost/Services/CostInvoiceService.php`
```php
$updates = array_filter([
    'work_order_title' => $data->workOrderTitle,
    'vendor' => $data->vendor,
    'bill_to' => $data->billTo !== null
        ? array_map(fn ($target) => $target->value, $data->billTo)
        : null,
    'attachment_name' => $data->attachmentName,
    'attachment_size' => $data->attachmentSize,
], fn ($value) => $value !== null);
```
- What's wrong: Nullable fields (`attachment_name`, `attachment_size`) can never be cleared via
  PATCH — `array_filter(... !== null)` strips every null before update.
- Why: `UpdateCostInvoiceRequest::toData()` uses `array_key_exists(...)` to preserve an explicit
  null (intent: allow clearing), but (a) the DTO collapses "absent" and "explicit null" into the
  same null, and (b) this service filters all nulls out. The FormRequest's array_key_exists code
  is effectively dead. Same defect in `CostLineItemService::update()` lines 41-46 (`description`).
- Fix: Make request + DTO + service agree on one contract. Either drop the array_key_exists
  handling if clearing is out of scope, or drive `$updates` off present request keys so an
  explicit null persists.

**File:** `app/Modules/OperativeCost/Services/CostLineItemService.php`
```php
private function lineItemToArray(CostLineItem $lineItem): array
{
    return [
        'id' => $lineItem->id,
        'cost_invoice_id' => $lineItem->cost_invoice_id,
        'item_type' => $lineItem->item_type->value,
        'description' => $lineItem->description,
        'qty' => round((float) $lineItem->qty, 2),
        'sale' => round((float) $lineItem->sale, 2),
        'total' => round((float) $lineItem->qty * (float) $lineItem->sale, 2),
    ];
}
```
- What's wrong: Line-item serialization + `round`/`qty*sale` math duplicated across both services,
  and the two copies produce different shapes for the same entity.
- Why: `CostInvoiceService::invoiceToArray()` (lines 141-148) builds the line item inline but
  OMITS `cost_invoice_id`, which this copy includes. A line item in `GET .../cost-invoices` differs
  from the same line item from `POST/PATCH .../line-items`. Code-review skill: extract copied logic.
- Fix: Extract one serializer (shared method / trait / model `toArray()`) + shared round/lineTotal
  helpers; decide once whether `cost_invoice_id` is part of the line-item contract.

### NIT

**File:** `app/Modules/OperativeCost/Services/CostInvoiceService.php`
```php
private function lineTotal(CostLineItem $item): float
{
    return (float) $item->qty * (float) $item->sale;
}
```
- What's wrong: `grand_total` can drift a cent from the sum of the per-line `total` values shown.
- Why: per-line `total` = `round(qty*sale, 2)`, but `grand_total` = `round(sum(qty*sale_unrounded))`.
  `qty`/`sale` are `decimal:2`, so the product can carry 4 decimals; round-then-sum vs
  sum-then-round can differ. Money on screen.
- Fix: Sum the already-rounded per-line totals for `grand_total`/`totals_by_type` so columns
  reconcile with the footer.

## Verdict (round 1)
needs changes before merge (no blockers; 2 should-fix + 1 nit)

## Re-review — commit 9ef1ead ("Fixed clearable cost invoice attributes and extracted line item serializer logic")
Both should-fix resolved:
- SHOULD-FIX #1 (clear nullable fields on PATCH): FIXED. Removed `toData()` from both Update
  requests; controllers now pass `$request->validated()` straight to `update(array $data)` →
  `$model->update($data)`. Omitted key (`sometimes`) = left alone; explicit null = cleared.
  Covered by 4 new tests (feature + unit, invoice + line item) asserting the clear behavior.
- SHOULD-FIX #2 (duplicated/divergent serializer): FIXED. Extracted public
  `CostLineItemService::toArray()`, reused by `CostInvoiceService` via DI. Embedded line items
  now include `cost_invoice_id` (shape matches the line-item endpoints). New test asserts the
  key is present and equals the invoice id. No circular dependency (both singletons;
  CostLineItemService has no deps).

Remaining after re-review:
- NIT (new, introduced by the fix): both service `update()` docblocks reference
  `UpdateCostInvoiceRequest::toData()` / `UpdateCostLineItemRequest::toData()`, a method this
  same commit deleted. Stale reference → point it at `$request->validated()` / `rules()`.
- NIT (pre-existing): grand_total rounding drift (sum-unrounded vs round-per-line). Unchanged.

## Verdict (round 2 — final)
ready to merge (0 blockers, 0 should-fix; 2 nits, non-blocking)

## SPACE log
- Time spent reviewing: 30 min (round 1)
- Johnny's read/validate time: 30 min
- Model: Opus 4.8
- Findings round 1: 0 blockers, 2 should-fix, 1 nit
- Findings round 2: both should-fix resolved; 2 nits remain (1 new stale-comment, 1 pre-existing)
- Verdict: ready to merge

## Review closed
- Timestamp: 2026-07-13 15:52 -03
- Resolution: merged
- Notes: Both should-fix (clearable nullable fields on PATCH; extracted shared line-item
  serializer) were addressed in commit 9ef1ead and merged. Two nits waived at merge time
  (stale `::toData()` docblock references in both services' update() methods; pre-existing
  grand_total rounding drift) — non-blocking, dev's discretion.
