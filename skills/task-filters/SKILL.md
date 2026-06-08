# Skill: Task Filters & Mapper System

## When to read this
Adding a new filter param, a new schedule bucket, or any change to how tasks are queried.

## The split — where each piece lives
- **SA filter params** (what the FE sends) → `TaskListData` DTO (`app/Modules/Tasks/DTOs/TaskListData.php`).
- **RM translation** (how those params become RM filter strings) → `RentManagerIssueMapper` (`app/Modules/Integrations/RentManager/Mappers/`).
- **Never** put RM PascalCase field names in `TaskListData` or `TaskService`.

## Adding a new SA filter param — step by step
1. Add a nullable property to `TaskListData`.
2. Add a validation rule to the task-list Request.
3. Map the request param → DTO in the `MapsTaskRequestToData` concern.
4. Add the RM translation in `RentManagerIssueMapper::buildFilters()`.
5. Add a test case in `RESTClient/tasks.http`.
6. Add a Feature test in `tests/Feature/Modules/Tasks/`.

## Schedule buckets (semantic filters)
Handled in `RentManagerIssueMapper::buildScheduleClauses()`:
- `overdue`: `IsClosed=false` + `DueDate <= yesterday`
- `scheduled`: `IsClosed=false` + `DueDate >= today`
- `pending`: `IsClosed=false` + in `StatusID = STATUS_PENDING` (39)
- `completed`: `IsClosed=true`

RM has no OR operator, so only ONE `schedule` value is accepted per request — this is enforced in `buildScheduleClauses()`. FE chip counts will NOT sum to the total because the buckets intentionally overlap.

## Multi-value filters (array params)
Handled by `RentManagerIssueMapper::buildInClause()`:
- Single value → `"Field,eq,X"`
- Multiple values → `"Field,in,(X,Y,Z)"`
- Linked collection fields use dotted paths → `"LinkedTenants.TenantID,in,(1,2)"`

## Count-only queries
- Use `RentManagerIssueMapper::countParams()` — `pageSize=1`, minimal fields.
- Read the total from `meta.pagination.total` in the RM response.
- Never fetch the full list just to count.

## Stats / cached counts
- `TaskStatsService` caches results in Redis.
- Bust the cache with `?refresh=1`.
- `STATUS_PENDING = 39` is a hardcoded constant in `RentManagerIssueMapper`. If it varies by environment, move it to config.

## Passthrough compatibility
SA accepts a raw RM `filters` string alongside the typed SA params. Both are merged in `buildFilters()` and appended as clauses — SA params take precedence in naming, but both are honored. This dual compatibility is intentional; don't "clean it up" by dropping the raw passthrough.
