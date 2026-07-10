# DEVSYM-376 — [BE] Internal user account management — API + data model
> Refined 2026-07-09. Source of truth for implementation is the refined Linear scope (pinned 2026-07-08 comment) + AGENTS.md, NOT the original Linear body.

## Current ask (as read from Linear)
SymAssist needs to create and manage internal-user (superuser) accounts — Jeremy (Portfolio Manager) + internal staff, scaling as SymAssist hires. Admin-only, not self-service. Account-lifecycle layer: create / list / edit / deactivate internal users, assign a role, email credentials on creation, force reset on first login. Identity is SymAssist-owned and fully decoupled from RM (global static Partner Token, no RM login, no credential reuse). This ticket makes NO RM calls.

## Target
Repo(s): BE  •  Module: NEW `app/Modules/InternalUsers/` (mirrors `app/Modules/UserProfile/`)  •  Layer(s): SA (local DB, snake_case)  •  RM-backed: no (zero RM calls)
Contract source: none (new SA surface — documented in the PR for DEVSYM-409 FE).
Relevant existing files (read, not touched): `app/Models/User.php`, `app/Models/UserProfile.php`, `app/Modules/UserProfile/*` (template), `app/Http/Middleware/EnsureIsSuperAdmin.php`, `config/rent_manager.php` (admin_user_id), `config/mail.php`, `bootstrap/providers.php`.

## Refined scope
Boundaries pinned by sibling tickets:
- Role DEFINITIONS + cross-PM access policy = **DEVSYM-373**. 376 only persists `role_id` and materialises the 5-role list; zero role semantics / permission gating / fan-out.
- Credential plumbing / `pm_connections` / context resolver = **DEVSYM-399** (merged foundation). 376 does not touch them.
- FE = **DEVSYM-409**. 376 is BE-only; documents every request/response shape in the PR.
- The original Linear body (3 roles, admin-set password, auto fan-out to all PMs) is **SUPERSEDED** by the 2026-07-08 refinement comment. Absorbs DEVSYM-372 (closed as duplicate).

IN scope:
- `internal_users` model + one forward migration: `id`, `full_name`, `email` (unique), `password` (nullable, hashed), `role_id` FK, `is_active` (default true), `force_password_reset` (default true), `last_login_at` (nullable), `timestamps`.
- `roles` table + migration (`id`, `name` unique), seeded IN the migration (idempotent) with the 5 roles DEVSYM-373 defines: Portfolio Manager, Maintenance Manager, Maintenance Manager Agent, Customer Service Manager, Customer Service Agent. `role_id` FK on `internal_users`.
- Admin-guarded endpoints (`['api','auth:sanctum','rm_super_admin']`, `rm_token.valid` dropped — no RM call): `GET /api/internal-users` (list with role + active), `POST /api/internal-users` (full_name, email, role_id), `PATCH /api/internal-users/{id}` (update — never password), `PATCH /{id}/deactivate`, `PATCH /{id}/activate` (reversible).
- Credential flow (Johnny's call, Option A): on create → `password` null, `force_password_reset=true`, email a signed expiring set-password link (`URL::temporarySignedRoute`, ~48h) — never a cleartext password. ONE public endpoint `POST /api/internal-users/{id}/set-password` (`signed` middleware only) sets the hashed password and clears the flag.
- Credentials email = first Mailable in the app, on stock Laravel `Mail`/`Notification`.
- Tests: shape test per endpoint + service unit test (RULES: `User::factory()->create()`, `withoutMiddleware(BypassAuthForDev)`, acting user with the admin `rm_user_id`). Factories for Role/InternalUser.
- Output shape: `{ id, full_name, email, role:{id,name}, is_active, created_at, updated_at }` — password/temp credential never serialized.

NOT in scope: role semantics/policy (373); cross-PM fan-out (373); credential/`pm_connections`/context work (399); FE (409); self-registration; any RM call; admin-set passwords; internal-user login/session (deferred — only set-password on first login is here); editing already-shipped migrations (one forward migration each).

## Comment ledger
One comment (2026-07-08, Jhonattan Campo): **SUPERSEDED** marker. Refined against the RM partner onboarding call + planning call with Lean. Resolutions incorporated: identity SymAssist-owned & RM-decoupled (Partner Token global/static, no RM login/credential reuse); roles are cosmetic SA labels, no permission gating ("todos los superusers ven todo"); no per-PM/per-property scoping (future ticket); RM privileges are Product-level not per-user; "PM API keys" → `pm_connections`; absorbs DEVSYM-372; scope split BE-here / FE-in-409. Nothing left open in the thread.

## Contradictions / open questions
Two ticket assumptions contradict the actual codebase — both resolved before implementation:

1. **"Reuse the existing mailer, do not build a new one" — there is NO mailer to reuse.** Grep: zero Mailables/Notifications, no `app/Mail`, no email views; `config/mail.php` is stock (`MAIL_MAILER` defaults to `log`). → The credentials email is the FIRST mail in the app, built on stock Laravel `Mail` (honors the spirit: not a bespoke mailer service). In dev it lands in the log until a real transport (SMTP/SES) is set in env — infra config, outside this ticket's code.

2. **"Force reset on first login" is IN scope, but there is no internal-user login path today** (current `AuthService::login` authenticates against RM's `AuthorizeUser`, and internal users are RM-decoupled). Combined with "never email a cleartext password", the email must carry a signed set-password link, not a password. → **→ Johnny**: resolved to Option A — 376 ships the invite link + one public `signed` set-password endpoint; full internal-user login/session is deferred to a future auth ticket. `InternalUser` stays a plain Model (not `Authenticatable`) to avoid a speculative guard.

## Architecture decisions that apply
- **Separate identity table is correct, not a shortcut.** `users`/`App\Models\User` is RM-coupled (rm_user_id, rm_token, rm_roles, linked_owner/tenant/vendor; login upserts by rm_user_id; `user_type` derived from RM roles) = external portal identity. SymAssist-owned decoupled staff identity MUST be a new `internal_users` table; reusing/extending `users` would violate the decoupling the ticket mandates.
- SA layer speaks snake_case, no RM field names, no RM calls (AGENTS.md cardinal rule) — trivially satisfied since 376 is pure local DB.
- No speculative abstractions (AGENTS.md): `InternalUser` plain Model until a login flow needs `Authenticatable`; roles are a flat lookup table, no role/permission engine (that's 373); reuse `rm_super_admin` middleware, do not build a new admin guard.
- New migration only, never edit shipped ones (RULES + AGENTS.md); seed roles inside the migration so every environment has the FK targets without a manual `db:seed`.
- Every endpoint gets a shape feature test (RULES hard requirement).

## Traps in play
- **`rm_super_admin` is the ONLY admin gate** (`EnsureIsSuperAdmin`): it compares `user()->rm_user_id` to `config('rent_manager.admin_user_id')` (default 393, `RM_ADMIN_USER_ID`). There is NO role/permission table behind admin authz — it is "are you literally this one RM user ID". The admin caller is an RM-backed `User`; internal users being managed are a different table. Reuse the middleware as-is; do not invent a roles-based admin check (that would drift into 373).
- **Local `roles` do NOT exist.** The `Users` module's `RoleController`/`RoleService` are RM remote-API passthroughs (`rentManagerService->getRoles/createRole`), not a local table — do not confuse the new local `roles` table with them, and do not wire 376's roles through RM.
- **`DEV_AUTH_BYPASS`/`BypassAuthForDev` wins over `actingAs()` and phpunit env** (RULES): feature tests hitting `rm_super_admin` need `withoutMiddleware(BypassAuthForDev::class)` and a persisted acting user whose `rm_user_id` matches the configured admin id, or a 401-expected test passes as 200 in Docker.
- **Deactivate = `is_active` boolean flag, reversible — NOT Laravel SoftDeletes, NOT a hard delete** (ticket explicit + AGENTS.md hard-delete trap).

## Drift / flags
- No PascalCase RM field names anywhere (pure SA). ✅
- `pm_connections` / `RentManagerContextResolver` / contexts / `PmConnection` (DEVSYM-399) confirmed out of scope — not read-for-write, only read to confirm the boundary. ✅
- ⚠️ Mail transport gap (see Contradictions #1) — credentials email works but lands in `log` until env transport is configured. Flag in the PR.
- ⚠️ Emailed set-password link is stateless (signed URL, no revocation) — acceptable for an invite; if revocation is later required, switch to a hashed one-time token (out of scope now).

## Suggested estimate
Linear estimate: 8 points. Reasonable — greenfield module (2 tables + seed, 2 models, service/controller/2 requests/DTO/resource/routes/provider), the app's first Mailable + view, a public signed endpoint, and full feature+unit test coverage with new factories. The two codebase contradictions (no mailer, no login path) were the real cost and are resolved up front.
