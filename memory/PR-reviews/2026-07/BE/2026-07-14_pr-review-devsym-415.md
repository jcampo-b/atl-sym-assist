## Review opened
- Timestamp: 2026-07-14
- Repo: BE
- Branch: feat/devsym-415-be-internal-user-authentication
- Base branch: dev
- Issue: DEVSYM-415

## Dev's stated focus
Login + logout for internal_users (Sanctum session), a new guard
(EnsureIsInternalUser) replacing rm_super_admin on the DEVSYM-374 Portfolio
route, and a migration widening personal_access_tokens.tokenable_id from bigint
to varchar to support InternalUser UUID PKs. First real (non-RM) auth path in
the project — max scrutiny on timing-safety, session invalidation on
deactivation, guard precedence, and migration safety against existing data
(not just layer discipline / code quality).

## Files in scope
RESTClient/internal-users.http
app/Http/Middleware/EnsureIsInternalUser.php
app/Models/InternalUser.php
app/Modules/InternalUsers/Controllers/InternalUserAuthController.php
app/Modules/InternalUsers/InternalUsersServiceProvider.php
app/Modules/InternalUsers/Requests/LoginInternalUserRequest.php
app/Modules/InternalUsers/Services/InternalUserAuthService.php
app/Modules/InternalUsers/routes.php
app/Modules/Superuser/routes.php
bootstrap/app.php
database/factories/InternalUserFactory.php
database/migrations/2026_07_15_000001_widen_personal_access_tokens_tokenable_id_to_string.php
tests/Feature/Modules/InternalUsers/InternalUserAuthControllerTest.php
tests/Feature/Modules/Superuser/PortfolioControllerTest.php

## Focus-axis results
- Timing-safety (login): PASS. Hash::check runs on every branch (unknown email,
  password-never-set, wrong password) against DUMMY_HASH ($2y$12$, cost 12 =
  framework default; no config/hashing.php override). `$user?->password`
  assigned to a var before `?? DUMMY_HASH`, per RULES.md. No short-circuit skips
  the hash. No enumeration leak.
- Session invalidation on deactivate: PASS. EnsureIsInternalUser resolves
  $request->user() from DB per request and re-reads is_active fresh, so
  deactivation takes effect next request without token revocation. Both guarded
  routes (portfolio, logout) carry the internal_user guard; a deactivated user's
  token is inert. Covered by test (inactive factory).
- Guard precedence: PASS. Order ['api','auth:sanctum','internal_user']; Sanctum
  resolves polymorphically first, guard does instanceof InternalUser. Legacy
  RM-coupled User → 403. Error envelope matches EnsureIsSuperAdmin ({error,message}).
- Migration safety: up() PASS (verified in RULES.md against real pg_dump).
  down() is the one gap — see findings.

## Findings

### should-fix — RESOLVED (re-review 2026-07-14, 2 rounds)
Round 1: dev amended the widen commit. down() now runs
`DELETE FROM personal_access_tokens WHERE tokenable_type = 'App\Models\InternalUser'`
BEFORE the ALTER, with a comment stating the internal-user session loss is a
deliberate rollback consequence. Verified: DELETE precedes ALTER; tokenable_type
matches Eloquent's getMorphClass() FQCN (no morph map configured); Postgres
standard_conforming_strings treats the backslashes as literal — matches stored value.

Round 2 (re-amend, migration commit now 163b524, branch HEAD 27ebc74): dev added
a forward-looking WARNING to the down() comment — if a morph map
(Relation::enforceMorphMap) is ever added, Sanctum stores the alias, this WHERE
silently matches zero rows, and the ALTER aborts again on leftover UUID ids;
comment says update the string to the alias. Correct and precise (names the
silent failure mode). Rigorously confirmed via per-file numstat that the
migration is the ONLY change across both rounds (399 -> 409 -> 415 insertions;
migration 37 -> 43 lines = the full +6 delta); the other 13 files are byte-for-
byte the originally reviewed versions. Nothing else snuck in.

Note: dev also reinforced the rule in .atl/memory/RULES.md (agent layer, outside
the BE repo diff — not part of the mergeable PR, but a correct reinforcement).

**File:** database/migrations/2026_07_15_000001_widen_personal_access_tokens_tokenable_id_to_string.php
**Snippet:**
```php
    public function down(): void
    {
        DB::statement('ALTER TABLE personal_access_tokens ALTER COLUMN tokenable_id TYPE BIGINT USING tokenable_id::BIGINT');
    }
```
**What's wrong:** down() casts varchar->bigint without filtering UUID rows. Once
any InternalUser has logged in, its tokenable_id is a UUID; tokenable_id::BIGINT
on that row throws Postgres 22P02 and the rollback fails.
**Why:** The up() comment documents the column now holds BOTH bigint (users) and
UUID (internal_users) ids. down() assumes everything stored is numerically
castable — contradicts the invariant up() just introduced. Against the explicit
"migration safety against existing data" focus.
**Suggested fix:** Purge non-numeric tokens first, e.g.
`DELETE FROM personal_access_tokens WHERE tokenable_type = 'App\Models\InternalUser'`
before the ALTER, with a comment that rollback drops internal-user sessions by
design.

### Notes (not findings)
- up() uses ALTER COLUMN TYPE (Postgres ACCESS EXCLUSIVE + table rewrite);
  harmless on the small personal_access_tokens table, worth remembering if it grows.
- Each login mints a fresh 'api' token without revoking prior ones — identical
  to the legacy AuthService (multi-device sessions). Intentional, consistent.
- Service uses abort(401/403) with generic messages instead of the legacy
  AuthService's AuthenticationException; deliberate to avoid leaking which check
  failed. Fine.

## Verdict
ready to merge (forward path correct and safe; the single item is rollback
robustness, not a forward blocker)

## SPACE log
- Time spent reviewing: TBD
- Model: Opus 4.8
- Findings: 0 blockers, 1 should-fix, 0 nits
- Verdict: ready to merge
