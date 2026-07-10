# DEVSYM-373 — [BE] Role abstraction + cross-PM access policy + fan-out — repo recon

> Read-only recon on `dev` (SymAssist-Backend), 2026-07-09. Fills the gaps NOT already
> covered by the DEVSYM-374 diagnosis (same folder). References that doc; does not repeat it.
> Repo state: on `dev`, clean, `HEAD...origin/dev = 0/0` (up to date). 373 confirmed unstarted.

## NOTE on tooling
Bash output in this repo is passing through a token-redaction filter that rewrites certain
identifiers to `n`/`ln` (e.g. `Sanctum`, `super_admin`, `admin_user_id`, `role_id`, `active`).
All signatures below were verified via the Read tool (unfiltered), not Bash.

---

## G1 — Role abstraction: is there ANY SA-owned role construct today?
**NO SA-owned role enum, constant, table, or model exists. 373's role abstraction is GREENFIELD.**

- Migrations (complete list, 6 total): `create_users_table`, `create_cache_table`, `create_jobs_table`,
  `create_personal_access_tokens_table`, `create_user_profiles_table`, `create_pm_connections_table`.
  → **No `roles` migration. No roles table.**
- `app/Modules/Users/Services/RoleService.php:11` — **pure RM passthrough**, confirmed NOT an SA-local store:
  - `create(RoleData): array` → `RentManagerService::createRole()` + `attachRolePrivilege()` (RM `POST Roles` + `POST Roles/{id}/Privileges`).
  - `list(RoleListData): array` → `RentManagerService::getRoles()`, mapped by `RentManagerRoleMapper`.
- `app/Modules/Users/Controllers/RoleController.php:11` — thin controller over the passthrough (`index`/`store`).
- `app/Modules/Users/DTOs/RoleData.php`, `RoleListData.php` — RM role shells (name/isActive/privileges). NOT superuser roles.
- `RentManagerRoleService`, `RentManagerRoleMapper` (Integrations/RentManager) — RM-remote.
- `app/Models/User.php` carries RM-coupled `rm_roles`/`rm_privileges` array-cast columns, populated from RM
  `UserRoles`/privileges in `AuthService`. RM concept, not an SA role store.
- **Only native PHP enum in `app/`** is `app/Modules/Tasks/StateMachine/TaskStatus.php` (`enum TaskStatus: string`).
  There is NO `Role` enum.

**Bottom line:** 373 creating an `enum Role` overlaps with **nothing**. The roles *table* / `internal_users` /
`role_id` FK belong to 376 (unbuilt); 373's enum is the value type and is orthogonal.

---

## G2 — Superuser identity at 373-time
**Confirmed: NO `internal_users` table, NO `role_id` DB column, NO non-RM SA-owned auth path.**

- No `internal_users` migration (see G1 list). The `role_id`/`ns` grep hits are RM role-assignment *payload*
  fields in Users/Onboarding DTOs & requests, **not** DB columns.
- Superuser is identified end-to-end by RM identity only:
  - `app/Http/Middleware/EnsureIsSuperAdmin.php:13` →
    `if ($request->user()?->rm_user_id !== config('rent_manager.admin_user_id')) → 403`.
  - `config/rent_manager.php` → `'admin_user_id' => env('RM_ADMIN_USER_ID', 393)`.
  - Registered as middleware alias `rm_super_admin` (used on `POST /users/roles`, onboarding routes, etc.).
- Auth is RM-backed only: `User` uses Sanctum `HasApiTokens`; `AuthService` authenticates against RM and mints a
  Sanctum token; `me`/`logout` are `auth:sanctum`. `BypassAuthForDev` just loads the RM admin `User` by `rm_user_id`.
  **No Sanctum/auth path authenticates a non-RM, SymAssist-owned user.**

**Bottom line:** YES — 373 can be built and unit-tested as **stateless domain logic** (role value in →
connections/clients out) with **no** dependency on a logged-in user or `internal_users`.

---

## G3 — PmConnection querying
**"Reach all active connections" needs a NEW scope/query built from scratch.**

- `status` enum values (migration `2026_07_01_000001...:16`): `['pending', 'active', 'suspended']`, default `'pending'`.
- `app/Models/PmConnection.php`: plain Eloquent model. **No scopes** (`scopeActive` absent), **no relations**
  (none to user/role/location), status is an un-cast plain `string`. `$casts`: `rm_location_id`=integer,
  `rm_credential_ref`=encrypted, `onboarded_at`=datetime.
- **No `->where('status', ...)`, no `PmConnection::where/all/get/query` anywhere in `app/`.** Zero existing
  active-connection query.

**Bottom line:** greenfield — 373 must introduce the "active connections" query/scope itself.

---

## G4 — Fan-out entry point + return type
- Entry point confirmed: `RentManagerContextResolver::clientForConnection(PmConnection $pmConnection): RentManagerHttpClient`
  (`.../Context/RentManagerContextResolver.php:21`). Body:
  `$this->defaultClient->withContext($this->contextForConnection($pmConnection))`.
  `RentManagerHttpClient::withContext()` returns `new self($context)` → a fresh, isolated per-connection client (safe to fan out).
- **Single scoped-client GET return type** — `RentManagerHttpClient::get(string $endpoint, array $query = []): array`
  (`RentManagerHttpClient.php:134`). Returns `array<string, mixed>` = **raw decoded JSON associative array**
  (`[]` on empty body; `['value' => $decoded]` if JSON is non-array). **No DTO, no pagination wrapper.**
- Contract alternatives if headers/pagination are needed:
  - `getResponse(string, array = []): Response` (line 142) — full body + headers.
  - `poolGet(string $endpoint, array $requests): array` / `pool(array): array` (lines 71 / 92) — concurrent GETs,
    return `array<string, Response|ConnectionException>`, keyed by request key, with pooled-401 refresh-and-refire.
- **Zero callers** of `clientForConnection`/`contextForConnection` outside the resolver class (grep empty).
  **No code iterates multiple connections/clients anywhere.** (Matches the 374 diagnosis; still true.)

**Bottom line (373→374 contract):** a single scoped-client fetch yields a raw PHP `array` (decoded JSON).
Fan-out collects one `array` per reached connection. Use `getResponse(): Response` instead only if 374 needs
RM headers/pagination cursors. `poolGet`/`pool` already exist if fan-out wants concurrency within one client —
but cross-connection fan-out means N *different* clients, so the loop is 373's to build.

---

## G5 — Onboarding / status transitions
**No code sets `pm_connections.status` to `'active'`.**

- Only writers of `status`: the migration default (`'pending'`) and tests constructing `PmConnection` instances
  directly (`tests/Unit/.../Context/PmConnectionContextTest.php`).
- The Onboarding module does **not** reference `PmConnection` at all. No seeder, no factory, no admin endpoint
  activates a connection.

**Bottom line:** 373's "onboarding hook" is **NOT** code to write here. A live `where status = 'active'` query
auto-includes any future activated PM with nothing to hook into. Activation flow belongs elsewhere
(onboarding / 376 / 399), not 373.

---

## G6 — SA module template
- **Canonical CRUD-style SA module skeleton:** `app/Modules/{Name}/` with
  `Controllers/` · `DTOs/` · `Requests/` (FormRequests) · `Services/` · `{Name}ServiceProvider.php` · `routes.php`.
  Concrete example: `app/Modules/Users/` (Controller → FormRequest `toData()` → Service singleton → DTO;
  `UsersServiceProvider` binds services as singletons; `routes.php` groups under `auth:sanctum`+`rm_token.valid`).
- **Domain-machinery precedent (the right mirror for role enum + policy + fan-out):**
  `app/Modules/Tasks/StateMachine/` —
  - `TaskStatus.php` → `enum TaskStatus: string` with `match`-based methods (`internalId()`), RM mapping kept OUT
    of the enum. **This is the exact shape a `Role` enum should follow.**
  - `TaskStateMachine.php`, `Rules/TransitionRule.php` + `Rules/TransitionRuleEngine.php`, `Exceptions/` — the
    policy-engine + rules layout that a "cross-PM access policy" naturally mirrors.
- Fan-out service placement options: a new SA module (e.g. `app/Modules/Superuser/Services/`) mirroring the Users
  skeleton, or under `Integrations/RentManager/Services/` since it wires the existing resolver. Enum-ish catalog
  precedent also exists at `Integrations/RentManager/Catalogs/` (`RentManagerPrivileges`).

---

## BOTTOM LINE (explicit answers)
1. **373 = fully stateless domain machinery (enum + policy + fan-out), buildable & unit-testable WITHOUT
   internal_users(376) or login(auth)?** → **YES.**
2. **Concrete return type of a single scoped-client fetch (373→374 contract)?** → raw PHP `array` (decoded JSON
   associative) from `RentManagerHttpClient::get(): array`; optionally `Response` via `getResponse()` if headers/pagination needed.
3. **Does any roles table/enum already exist (overlap risk with 376)?** → **No SA-owned one.** Only an RM-passthrough
   `RoleService`, RM `rm_roles` cast on `User`, and one unrelated `TaskStatus` enum. Greenfield → **no overlap** with 376.
4. **Is the "onboarding hook" real code or free from a live query?** → **Free.** A live `where status='active'`
   query satisfies it; nothing to hook into today.
