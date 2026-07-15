## Review opened
- Timestamp: 2026-07-14
- Repo: BE
- Branch: feat/devsym-376-be-internal-user-account-management
- Base branch: dev
- Issue: devsym-376

## Dev's stated focus
BE account-lifecycle layer for SymAssist internal staff (DEVSYM-376). New local `roles` +
`internal_users` tables, admin CRUD (list/create/update/deactivate/activate) guarded by
`rm_super_admin`, and an invite-by-signed-link credential flow (public, signed set-password
endpoint + first Mailable). Identity is SymAssist-owned, fully decoupled from RM (no rm_user_id,
no RM login, zero RM calls). Login/session deferred to DEVSYM-415.

## Files in scope
RESTClient/internal-users.http
app/Mail/InternalUserCredentialsMail.php
app/Models/InternalUser.php
app/Models/RoleRecord.php
app/Modules/InternalUsers/Controllers/InternalUserController.php
app/Modules/InternalUsers/DTOs/InternalUserData.php
app/Modules/InternalUsers/InternalUsersServiceProvider.php
app/Modules/InternalUsers/Requests/SetPasswordRequest.php
app/Modules/InternalUsers/Requests/StoreInternalUserRequest.php
app/Modules/InternalUsers/Requests/UpdateInternalUserRequest.php
app/Modules/InternalUsers/Services/InternalUserService.php
app/Modules/InternalUsers/routes.php
bootstrap/providers.php
database/factories/InternalUserFactory.php
database/migrations/2026_07_14_000001_create_roles_table.php
database/migrations/2026_07_14_000002_create_internal_users_table.php
resources/views/mail/internal-users/credentials.blade.php
tests/Feature/Modules/InternalUsers/InternalUserControllerTest.php
tests/Feature/Modules/InternalUsers/SetPasswordControllerTest.php

## Findings

### Should-fix

**File:** `database/migrations/2026_07_14_000001_create_roles_table.php`
**Snippet:**
```php
        // `roles` is a lookup table an FK depends on (structure, not business
        // data), so the catalog is seeded here — idempotently — rather than in
        // a data seeder. Role (the enum, DEVSYM-373) is the single source of
        // truth; this only materializes it. `name` is the enum's backing value
        // (snake_case, e.g. "portfolio_manager").
        $now = now();

        foreach (Role::cases() as $role) {
            DB::table('roles')->updateOrInsert(
                ['name' => $role->value],
                ['created_at' => $now, 'updated_at' => $now],
            );
```
- What's wrong: role catalog materialized only inside this one-time create migration; a future
  `Role` enum case won't propagate to environments where this migration already ran.
- Why: same failure mode as RULES.md "migrations don't re-run" rule — new `Role::cases()` entry
  silently missing its `roles` row → `exists:roles,id` validation + `role_id` assignment fail in
  staging/prod while passing on a fresh DB.
- Fix: keep first-run seed (FK ordering justifies it) but make explicit (comment or convention)
  that adding a role needs a follow-up seeding migration, or move the loop to an idempotent
  deploy-run seeder.

### Nit

**File:** `database/factories/InternalUserFactory.php`
**Snippet:**
```php
            'role_id' => RoleRecord::query()->value('id')
                ?? RoleRecord::create(['name' => Role::PortfolioManager->value])->id,
```
- What's wrong: `?? RoleRecord::create(...)` fallback is unreachable — `RefreshDatabase` runs the
  roles migration which always seeds the catalog, so `value('id')` is never null.
- Why: code-quality rule — no dead/defensive code for an impossible condition.
- Fix: simplify to `RoleRecord::query()->value('id')` (or `inRandomOrder()->value('id')` for
  role variety).

## Notes (verified OK — not findings)
- `test_store_requires_super_admin` uses `config(['app.dev_auth_bypass' => false])` instead of
  `withoutMiddleware(BypassAuthForDev::class)`. Valid equivalent: `BypassAuthForDev` gates on
  `config('app.dev_auth_bypass')` in its own `if`, so toggling the config disables it. Also the
  middleware requires `app()->environment('local')`, and in Docker APP_ENV=local leaks past
  phpunit's non-forced `<env>`, which is why the bypass matters at all.
- Layer discipline clean: zero RM PascalCase, zero RM calls in the module.
- `rm_super_admin` without `rm_token.valid` on admin routes is correct (documented in routes.php);
  SA-owned identity makes no RM call. (Note: internal_users can't pass `rm_super_admin` today —
  they have no rm_user_id — but login/guard is explicitly DEVSYM-415 scope, not this PR.)
- Endpoint coverage complete: index/store/update/deactivate/activate + signed set-password
  (including reused-link 410, unsigned 403, confirmation) all have feature tests.
- `password` cast `hashed` + `$hidden` + DTO whitelist = password can't leak and is hashed on set.
- Migration order (roles 000001 → internal_users 000002) satisfies the FK dependency.

## Verdict
ready to merge — no blockers; 1 should-fix (forward-compat comment/convention on role seeding),
1 nit (dead factory fallback).

## SPACE log
- Time spent reviewing: TBD
- Model: Opus 4.8
- Findings: 0 blockers, 1 should-fix, 1 nit
- Verdict: ready to merge
