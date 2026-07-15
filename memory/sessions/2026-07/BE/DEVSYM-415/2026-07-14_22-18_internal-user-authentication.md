# DEVSYM-415 — [BE] Internal user authentication (login + role-aware guard)

**Date:** 2026-07-14
**Branch:** `feat/devsym-415-be-internal-user-authentication` (base `dev`)
**Priority:** High. Blocked-by DEVSYM-376 (merged, PR #142). Related: 373/374/409.

## What was done

Built the login + guard that DEVSYM-376 deferred. `internal_users` can now log
in (email + password → Sanctum token) and a new `internal_user` middleware
guards the Superuser stack, replacing the RM-coupled `rm_super_admin` on
DEVSYM-374's Portfolio route. No role gating (373 keeps `Role` inert for reach).
Zero RM calls — internal identity stays fully decoupled.

## Files touched

**Model / auth enablement**
- `app/Models/InternalUser.php` — now extends `Illuminate\Foundation\Auth\User` + `HasApiTokens` (was plain `Model`). No `config/auth.php` change: Sanctum resolves the tokenable polymorphically, so `auth:sanctum` loads an `InternalUser` from its own token.

**Login (module `app/Modules/InternalUsers/`)**
- NEW `Controllers/InternalUserAuthController.php` (login + logout, thin).
- NEW `Services/InternalUserAuthService.php` (auth separate from admin CRUD, mirrors the Auth module split).
- NEW `Requests/LoginInternalUserRequest.php` (email + password).
- `routes.php` — `POST /api/internal-users/login` (public, `throttle:10,1`), `POST /api/internal-users/logout` (`auth:sanctum` + `internal_user`).
- `InternalUsersServiceProvider.php` — singleton the new service.

**Guard**
- NEW `app/Http/Middleware/EnsureIsInternalUser.php` (alias `internal_user`): `instanceof InternalUser && is_active`. NOT role-aware.
- `bootstrap/app.php` — registered the alias.
- `app/Modules/Superuser/routes.php` — Portfolio guard swapped `rm_super_admin` → `internal_user`; coexistence documented in the route comment.

**Migration**
- NEW `database/migrations/2026_07_15_000001_widen_personal_access_tokens_tokenable_id_to_string.php` — widens `tokenable_id` bigint → `VARCHAR(255)` (raw `ALTER ... USING`, no dbal).

**Tests / infra / docs**
- NEW `tests/Feature/Modules/InternalUsers/InternalUserAuthControllerTest.php` (6 tests).
- `tests/Feature/Modules/Superuser/PortfolioControllerTest.php` — retargeted to `InternalUser`; `withoutMiddleware(BypassAuthForDev)` in setUp.
- `database/factories/InternalUserFactory.php` — `withPassword()` + `inactive()` states.
- `RESTClient/internal-users.http` — login (happy + 401 + 403), Portfolio via internal token, logout.

## Decisions made

- **No `config/auth.php` guard/provider added.** Sanctum resolves the owning model via `tokenable_type`; a second Authenticatable + `HasApiTokens` model consumes Sanctum tokens with no named-guard entry. Avoids over-engineering.
- **Guard replaces, does not OR.** Portfolio now needs an active `internal_user`; `rm_super_admin` stays on RM-coupled routes (Users, Onboarding, Properties, RentManager, InternalUsers admin CRUD). RM admin bootstraps the first internal_users; they then log in for Portfolio. Coexistence is documented, per the ticket trap. (Johnny approved "replace" over "accept both".)
- **Login failure order:** verify credentials first (generic 401, no email enumeration) → then `is_active` (403). `is_active` re-checked in the guard so a live token can't outlive a deactivation.
- **Timing-safe login:** always run `Hash::check` against a constant `DUMMY_HASH` when there is no real hash (unknown email / password never set), so a fast-fail can't leak account existence via response timing. Measured: success 264.7 ms vs failures 270–274 ms — same order, no leak.
- **Login + logout** both shipped (Johnny approved logout despite AC listing only login).
- Commit units: model / login / guard+wiring / migration / tests.
- **Opus PR-review should-fix applied:** migration `down()` now `DELETE`s `WHERE tokenable_type = 'App\Models\InternalUser'` BEFORE the `ALTER ... TYPE BIGINT`, else UUID token ids abort the cast (`22P02`). FQCN string verified against `createToken()` (no morph map). Amended into the migration commit (`cde9e15`, was `a952db1`) via `--fixup` + non-interactive autosquash rebase; nothing was pushed. Rollback proven on a dev copy: `User`#393 token survives, UUID `InternalUser` token purged, column back to bigint.

## Verification

- Pint pass; PHPStan L5 no errors; 9 new/updated tests pass.
- Full suite = **22 pre-existing failures identical to clean `dev`** (confirmed by stashing WIP and re-running: 22 failed on dev vs 22 failed + 6 new passing with the branch). All 22 are the `DEV_AUTH_BYPASS` 401-vs-200 gotcha + RM-mock flakies, none in touched modules.
- **Migration proven against a `pg_dump` copy of dev** with a real preexisting `User` admin token: `artisan migrate` (real file) flipped `tokenable_id` bigint → varchar(255), token row `md5` byte-identical before/after (data intact), and a UUID `InternalUser` token — which previously failed Postgres `22P02` — now inserts.

## Open questions

- None blocking. FE login screen is a separate ticket (out of scope). Per-role gating is a future ticket (373 keeps role inert). `PortfolioCacheService::CALLER_ROLE_PLACEHOLDER` intentionally NOT touched — belongs to the per-role ticket.
- Pending: `pr-review-symassist` (Opus) over the full branch before push (Johnny running it). PR not yet opened.
