# Skill: New Laravel Module

## When to read this
Any time you create a new module under `SymAssist-Backend/app/Modules/`.

## Module folder structure (required)
```
app/Modules/{ModuleName}/
├── Controllers/
├── DTOs/
├── Events/          (create only if needed)
├── Jobs/            (create only if needed)
├── Models/          (create only if needed — local DB entities)
├── Repositories/    (create only if needed)
├── Requests/
│   └── Concerns/    (shared validation rules)
├── Services/
├── {ModuleName}ServiceProvider.php
└── routes.php
```
Create only the folders you actually use. Don't scaffold empty `Events/`, `Jobs/`, etc.

## ServiceProvider checklist
- Must extend `Illuminate\Support\ServiceProvider`.
- `boot()` calls `$this->loadRoutesFrom(__DIR__ . '/routes.php')`.
- Register the provider in `bootstrap/providers.php` (append — do not replace anything).
- Do NOT register in `config/app.php` (that is the pre-Laravel 11 pattern).

## Routes file conventions
- Wrap routes in `Route::middleware(['api', 'auth:sanctum', 'rm_token.valid'])`.
- Prefix with `api`.
- Route names use dot notation: `module.resource.action`.
- Declare non-resource routes BEFORE `apiResource()` to avoid `{param}` collisions.

## DTO conventions
- `final readonly class`.
- Constructor promotion for all properties.
- All properties nullable with defaults — never force required at the DTO level (validation belongs in Requests).
- Suffix: `Data` (e.g. `TaskData`, not `TaskDTO`). See `app/Modules/Tasks/DTOs/` for live examples (`TaskData`, `TaskListData`, `SubtaskData`, `TaskStatsData`).

## Controller conventions
- Thin: validate in a Request, logic in a Service, shape in a DTO.
- Always return `JsonResponse`.
- Use OpenApi attributes (`#[OA\Get]`, `#[OA\Post]`, …) on every public endpoint.
- Never inject `RentManagerHttpClient` directly — inject the relevant Service.

## Request conventions
- Shared rules → `Concerns/` traits (`HasXRules`).
- `StoreXRequest` / `UpdateXRequest` use the same Concern.
- Always return a `rules()` array; never `authorize(): false` in production.

## After creating the module
- Add a `.http` test file: `RESTClient/{module-name}.http`.
- Add the module to the **Active modules** table in `.atl/AGENTS.md`.
- Run `php artisan route:list` to verify routes registered correctly.
- Write a memory entry in `.atl/memory/`.
