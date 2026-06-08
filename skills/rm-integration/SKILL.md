# Skill: RentManager Integration

## When to read this
Any time you add, modify, or debug anything in `SymAssist-Backend/app/Modules/Integrations/RentManager/`, or any SA service that calls it.

## The proxy flow (full chain)
Every call to RentManager travels this chain. Each class owns exactly one responsibility:

```
SA Service                       e.g. TaskService — speaks snake_case, knows nothing about RM
  → RentManagerService           orchestrates RM calls, picks the right mapper
    → RentManager*Service        per-resource RM service (uses PerformsRentManagerRequests trait)
      → RentManagerHttpClient    the ONLY class that issues HTTP to RRM
        → RentManagerAuthService supplies the cached token
          → RRM (RentManager API)
```

Translation (PascalCase ⇄ snake_case) happens in the **Mappers**, not in the services.

## Adding a new RM resource — step by step
1. **Check field names.** Open `SymAssist-Backend/docs/rent-manager/raw/` (`00-models.json`, `embeds-*.json`) for the correct RM PascalCase names. Do not guess.
2. **Create the Mapper** (`Mappers/RentManager{Resource}Mapper.php`). Static methods only. Match the existing naming in `RentManagerIssueMapper`:
   - `from{Resource}Data(DTO): array` — SA DTO → RM request body (PascalCase)
   - `to{Resource}(array $row): array` — raw RM array → SA response (snake_case)
   - `listParams(DTO): array` / `countParams(DTO): array` — build RM query params
3. **Create the Service** (`Services/RentManager{Resource}Service.php`). Use the `PerformsRentManagerRequests` trait; call through `RentManagerHttpClient`, never raw HTTP.
4. **Create the Controller** — thin: validate via a Request → call the service → return `JsonResponse`.
5. **Register the route** in `Integrations/RentManager/routes.php` under `/api/integrations/rent-manager/`.
6. **Add the OpenApi attribute** (`#[OA\Get]`, etc.) on the controller method.
7. **Add a `.http` example** to `RESTClient/integrations.http`.

## Mapper conventions
- Static methods only — mappers hold no state.
- Input: an SA DTO, or a raw RM array.
- Output: an RM request body (PascalCase) OR an SA response (snake_case).
- **All PascalCase RM field names live only in Mapper files.** Nowhere else.
- Strip empty fields before sending: `array_filter(fn ($v) => $v !== null && $v !== '')`.

## Filter string construction
- Format: `"Field,op,value;Field2,op2,value2"` (semicolon-separated clauses, AND-only).
- Operators: `eq`, `ne`, `gt`, `lt`, `ge`, `le`, `ct` (contains), `in`.
- Multi-value: `"Field,in,(1,2,3)"` — built by `RentManagerIssueMapper::buildInClause()`.
- Linked collection filter: `"LinkedTenants.TenantID,eq,123"` (dotted path).
- **NO OR operator.** And `schedule[]` accepts only ONE value per request.
- Always sanitize user input before interpolating it into a filter string.
- Reference: `SymAssist-Backend/docs/rent-manager-filters.md`.

## Embeds
- Format: `"Relation1,Relation2"`.
- ServiceManagerIssues: ALWAYS include `Status` — `StatusID` is not returned at the top level (it arrives as `ServiceManagerStatusID` inside the embedded `Status` relation).
- Use the `RentManagerIssueMapper::ALWAYS_EMBED` constant — never hardcode the `'Status'` string again. The mapper appends it automatically if missing.

## Auth / token
- `RentManagerAuthService::getToken()` checks Redis first and calls RRM only on a miss. TTL ~23h.
- Never call the RRM auth endpoint directly from any other class.
- Dev: `DEV_AUTH_BYPASS=true` skips Sanctum (`BypassAuthForDev` middleware). The RM token is still required.

## Error handling
- RRM errors are caught by `RentManagerHttpClient` and rethrown as `RentManager*Exception` (see `app/Exceptions/RentManager*.php`).
- Controllers should NOT catch RentManager exceptions — let Laravel's handler map them.
- Custom exceptions already map to the correct HTTP status (auth, 404, rate limit, …) automatically.
