# DEVSYM-376 — [BE] Internal user account management (API + data model)

**Date:** 2026-07-14
**Branch:** `feat/devsym-376-be-internal-user-account-management` (base `dev`)
**Estimate:** 8 points

## What was done

Built the SA-owned internal-user account lifecycle layer, fully decoupled from
RentManager (zero RM calls). New `InternalUsers` module mirroring the
`UserProfile` module template: admin-guarded CRUD (create / list / update /
deactivate / activate) plus a public, signed set-password endpoint. The
project's first Mailable delivers a signed, expiring, single-use set-password
link. `roles` lookup table is seeded idempotently from the DEVSYM-373 `Role`
enum; `internal_users` carries a `role_id` FK.

Implemented to the **refined** scope (5 roles, no fan-out, no admin-set
password, invite-by-signed-link) — the original Linear body was superseded by
the 2026-07-08 comment.

## Files touched

**Migrations**
- `database/migrations/2026_07_14_000001_create_roles_table.php` — `roles` (id, name unique, timestamps) + idempotent in-migration seed from `Role::cases()`.
- `database/migrations/2026_07_14_000002_create_internal_users_table.php` — UUID PK, full_name, email unique, password nullable, role_id FK, is_active, force_password_reset, last_login_at.

**Models**
- `app/Models/RoleRecord.php` — Eloquent lookup for `roles`. Named `RoleRecord` (NOT `Role`) to avoid colliding with `App\Modules\Superuser\Role` (the enum). Docblock states it is the table materialization of the enum, not a second source of truth.
- `app/Models/InternalUser.php` — HasUuids, `password` hashed + hidden, `role()` belongsTo RoleRecord (typed generic for PHPStan).
- `database/factories/InternalUserFactory.php`

**Module `app/Modules/InternalUsers/`**
- `InternalUsersServiceProvider.php` (registered in `bootstrap/providers.php`)
- `routes.php`, `Controllers/InternalUserController.php`, `Services/InternalUserService.php`
- `Requests/{Store,Update,SetPassword}InternalUserRequest.php`
- `DTOs/InternalUserData.php` (single place defining the public output shape — password never serialized)

**Mail**
- `app/Mail/InternalUserCredentialsMail.php` + `resources/views/mail/internal-users/credentials.blade.php` (first Mailable; lands in log until transport configured — DEVSYM-410).

**Tests + docs**
- `tests/Feature/Modules/InternalUsers/InternalUserControllerTest.php` (7 tests)
- `tests/Feature/Modules/InternalUsers/SetPasswordControllerTest.php` (4 tests)
- `RESTClient/internal-users.http`

## Decisions made

- **`internal_users` PK = UUID** (mirrors `user_profiles`); `{id}` is exposed in the public signed URL, so UUID reduces enumeration surface. `roles` uses auto-increment (`$table->id()`) — a small static lookup.
- **`roles.name` = enum backing value** (`portfolio_manager`, …), seeded straight from `Role::cases()` — no drift, no display labels in the table (FE humanizes). Confirmed with Johnny.
- **Single-use link** enforced via `force_password_reset`: `setPassword()` `abort_if(!force_password_reset, 410)`. Documented the assumption (one active link per user at a time) in `sendCredentialsEmail()` — revisit before adding a "resend link" feature.
- **Admin guard** = `rm_super_admin` reused; `rm_token.valid` dropped (no RM call). Matches the Superuser module precedent.
- **Auth deferred**: set-password does NOT log in or return a token (DEVSYM-415).
- Commit units: migrations+models / mail / module+wiring / tests.

## Verification

- Pint: pass. PHPStan level 5: no errors. Tests: **11 passed, 54 assertions**.
- `BypassAuthForDev` does not fire under env `testing` (guards on `environment('local')`), so tests use `actingAs(..., 'sanctum')` without `withoutMiddleware` — mirrors `PortfolioControllerTest`.
- PHPStan needed a typed `@return BelongsTo<RoleRecord, $this>` on `role()` and `@property` docblocks on `RoleRecord` to resolve `role->id`/`role->name`.

## Open questions

- None blocking. Mail transport is out of scope (DEVSYM-410); internal-user login/guard is DEVSYM-415.
- Pending: `pr-review-symassist` (Opus) over the full branch before push (Johnny running it).
