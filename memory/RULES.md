# RULES.md — Consolidated corrections and decisions
_Last consolidated: 2026-06-26. Add new rules here; do NOT add new correction files._

---

## RM token cache TTL

RM invalidates API tokens after 15 min of inactivity (or 24h absolute) —
confirmed against official RM API docs. Never cache an RM token with a TTL
longer than ~10 min; the 401+handleUnauthorized() retry is the safety net,
but a TTL that outlives RM's inactivity window forces avoidable 401s after
any idle gap. The TTL value lives in `config('rent_manager.token_cache_ttl_minutes')`
(default 10, `RM_TOKEN_CACHE_TTL_MINUTES` env override) as the single source
of truth — never hardcode a TTL per-consumer.

> **Rule:** Resolved in DEVSYM-400. All four consumers (`RentManagerAuthService`,
> `UserSessionContext`, `PmConnectionContext`, `AuthService::login()`) read
> `config('rent_manager.token_cache_ttl_minutes')`. If you add a new place
> that persists `rm_token_expires_at` or caches an RM token, read this config
> key — do not hardcode a new TTL literal.
> **Source:** `sessions/2026-07/BE/DEVSYM-400/2026-07-02_16-56_centralize-rm-token-cache-ttl.md`
> **Why:** This is exactly how the bug was introduced originally — the 23h
> value was copy-pasted into 4 separate places (one, `AuthService::login()`,
> wasn't even caught by the ticket that reported the other three).

> **Rule:** When writing a characterization test for a TTL/expiry value, use
> `Mockery::on()` asserting exact `Carbon` equality against `Carbon::setTestNow()`
> frozen time — do not test real cache expiry by advancing time. The test
> cache driver may be Redis, whose TTL is real wall-clock time and ignores
> `Carbon::setTestNow()`.
> **Source:** `sessions/2026-07/BE/DEVSYM-400/2026-07-02_16-56_centralize-rm-token-cache-ttl.md`
> **Why:** A test that "passes" by accident (real Redis TTL happens to not
> have expired yet in test runtime) is a false positive that won't actually
> catch a regression in the TTL calculation.

---

## RM Integration

> **Rule:** RM-layer methods that merge an `embeds` relation into caller-supplied `embeds` must guard the append with `str_contains($embeds, '<Relation>')` (mirror `RentManagerIssueMapper::listParams` `ALWAYS_EMBED`, lines ~124-128), never append unconditionally — otherwise a caller that already embeds it gets `embeds=Property,Property`. Applies even to a method with one caller today, since RM-layer service methods are public and reusable.
> **Source:** `sessions/2026-07/BE/2026-07-13_correction-pr-review-374-findings.md` (DEVSYM-374 PR #141, finding 3)
> **Why:** `RentManagerUnitService::getActiveUnits()` shipped `$existingEmbeds.'Property'` with no dedup; a duplicate embed is silently wrong and diverges from the codebase's one established embed-merge pattern.

> **Rule:** RM has no `IsActive`/`IsDeleted`-equivalent field documented for Tenant or Owner (checked `model-Tenant.json` and every Owner model doc — none exists; `Tenant.Status` is an undocumented raw int, not a filterable active flag). Property has `IsActive` (RM-filterable server-side, `IsActive,eq,true`); Units inherit it only via the embedded `Property.IsActive` (no unit-level field) — see `RentManagerUnitService::getActiveUnits()`. Do not invent a Tenant/Owner deleted/stale filter without an explicit, confirmed field — a wrong guess silently hides real records.
> **Source:** `sessions/2026-07/BE/DEVSYM-374/2026-07-13_cross-pm-caching-layer.md`
> **Why:** DEVSYM-374's ticket traps asked to "filter deleted/stale records" for all four slow-tier resources (properties, units, tenants, owners); only two of the four have a documented, safe filter criterion. Confirmed with Johnny rather than guessing on Tenant/Owner.

> **Rule:** RM omits `Hours` from the default ServiceManagerIssue response entirely — unlike DueDate/ScheduledDate, it is never present unless explicitly requested via `fields=...,Hours,...`.
> **Source:** `2026-06-18_20-00_devsym-345-estimated-time-minutes-be.md`
> **Why:** Prevents `estimated_time_minutes` from silently mapping to null despite correct mapper logic. Use `RentManagerIssueMapper::mergeDetailFields()` + `DETAIL_FIELDS` constant (maintenance contract: add every new mapper field here).

> **Rule:** Use `?? null` (not `?? ''`) as the fallback for `Title` when pre-fetching a raw issue before an RM update; `array_filter` will drop the null key and RM keeps its existing value.
> **Source:** `2026-06-10_pr-review-devsym-338.md`
> **Why:** `?? ''` sends `Title: ''` which triggers `"Issue cannot be blank."` from RM — the same error the original fix was trying to avoid.

> **Rule:** Auto-transition responses reflect the pre-auto-transition status; the FE must always `GET` after `PATCH /tasks/{id}/status` to get the final state.
> **Source:** `2026-06-05_pr-description-lessons.md`
> **Why:** The PATCH response is assembled before the rule engine fires, so the status in the body can be stale.

> **Rule:** `RM_DEFAULT_USER_LOCATION_ID` must be set in `.env`; if missing or `0`, `assignUserLocation` sends `0` and RM returns a 500.
> **Source:** `2026-06-09_00-00_devsym-322-post-users-500.md`
> **Why:** `(int) null` is also 0, so even a missing key hits the same failure silently.

> **Rule:** RM persists the user row via `AddUser` before `assignUserRole`/`assignUserLocation` run — a failure in the subsequent calls leaves an incomplete user with the username consumed; re-running the create will return a duplicate error.
> **Source:** `2026-06-09_00-00_devsym-322-post-users-500.md`
> **Why:** There is no transaction wrapping the three RM steps; partial failure is permanent in RM.

> **Rule:** The RM per-user token (`users.rm_token`) silently falls back to the global system token after 23h in `RentManagerHttpClient::client()` — wrong audit attribution and permissions after expiry.
> **Source:** `2026-06-10_13-15_devsym-305-auth-token-ttl.md`
> **Why:** This is a latent bug; the 23h `rm_token_expires_at` check is not re-authing the user, just degrading to 4MK credentials.

> **Rule:** `crc32()` can return a negative integer on 64-bit PHP; use `abs(crc32($value))` or `sprintf('%u', crc32($value))` for any cache key or identifier derived from it.
> **Source:** `PR-reviews/2026-06-19_pr-review-sym-329.md`
> **Why:** Negative cache keys like `tasks_ver_-1234567890` are legal but confusing and can cause collisions depending on cache backend.

> **Rule:** Any fan-out / loop over N independent RM targets (connections, tenants, vendors) must isolate each iteration in `try/catch (\Throwable)`, log sanitized (`exceptionClass` = `$e::class` + `exceptionMessage` only — NEVER the full Throwable, RM bodies carry PII), skip the failed target, and accumulate partial results. One broken target must not abort the rest. Mirror `OnboardingService::initTasks()`.
> **Source:** `2026-07-10_19-00_correction-fanout-per-item-isolation.md`
> **Why:** `PmConnectionContext::decodeCredentials()` throws `RentManagerAuthenticationException` on a null/malformed `rm_credential_ref`; without per-item isolation in `CrossPmFanout::fanOut()`, one bad PM connection aborts the entire superuser cross-PM view.

> **Rule:** ivfflat pgvector indexes built on empty tables yield 0 centroids and every similarity search returns 0 rows; run sequential scans (`seq-scan`) until the table reaches 5,000+ rows, then add `ivfflat` with `lists = sqrt(n)`.
> **Source:** `2026-06-23_19-28_voice-caller-architecture-discovery.md`
> **Why:** ivfflat centroids are computed at index-creation time; empty → 0 centroids → 0 results always.

> **Rule:** The RentManager Partner Token is bound to the whitelisted static egress IP (DEVSYM-394). From any non-whitelisted origin — a laptop, ngrok, CI — RRM answers HTTP 401 with `Authentication Failed - You are not authorized to access from your IP address.` and `ErrorCode: -2146233087`. This is permanent, not a pending item. Any local capability that depends on a Partner Token call must be served by a fake bound in the container, never by a real call.
> **Source:** DEVSYM-556.

---

## Layer discipline

> **Rule:** `poolVacancy` RM service specs must include a `map` callable applying the appropriate `RentManagerVacancyMapper` method — raw RM PascalCase rows must never reach the SA `VacancyService`. [verify-duplicate]
> **Source:** `PR-reviews/2026-06-19_pr-review-sym-329.md`
> **Why:** `poolDashboard` already follows this pattern correctly; inconsistency caused the SA layer to call RM mapper methods directly, violating the cardinal layer rule.

> **Rule:** Filtering on a PascalCase RM field reached via an `embeds` relation (not just top-level fields) must live in a method inside `app/Modules/Integrations/RentManager/` (Service or Mapper), never inline in an SA-layer caller — even when that method has only one caller today. The "confirm >1 caller before adding a service method" rule does NOT override this; layer discipline wins when the two conflict.
> **Source:** `sessions/2026-07/BE/2026-07-13_correction-embedded-field-filter-leak.md`
> **Why:** `PortfolioCacheService` (DEVSYM-374) first read `$unit['Property']['IsActive']` directly in the Superuser (SA) module to filter out units of inactive properties — an easy-to-miss variant of the cardinal-rule violation since the PascalCase field arrived via an embed, not a top-level response field. Fixed via `RentManagerUnitService::getActiveUnits()`.

> **Rule:** Environment is a detail of the outermost ring. No service, mapper or state machine calls `app()->environment()`. If a capability applies to one environment only, it is provided by an implementation bound in the container, and the enabling point is the service provider. An environment conditional inside domain code is a seam whose only decision point is the text editor — which is why `DO NOT COMMIT` comments do not prevent commits.
> **Source:** DEVSYM-556.

---

## Within-layer responsibilities

> **Rule:** Never duplicate a TODO in both the docblock and the method body — keep exactly one, inside the method body.
> **Source:** `2026-06-04_16-00_correction-comments.md`
> **Why:** Duplicate TODOs drift out of sync and add noise without value.

> **Rule:** An empty class body does not need a comment placeholder.
> **Source:** `2026-06-04_16-00_correction-comments.md`
> **Why:** The class declaration already communicates intent; a placeholder comment adds noise.

> **Rule:** Input validation for emergency transitions (e.g. `is_emergency` field constraints, allowed target status) belongs in `UpdateTaskStatusRequest`, not the Service — the Service should receive a typed value object (`EmergencyContext`).
> **Source:** `2026-06-08_correction-pr-review-152.md`
> **Why:** Mixing input validation with business logic makes the Service responsible for two layers.

---

## Git & PR workflow

> **Rule:** PR description body: 4-5 bullets max under "Changes Made"; no implementation details — those belong in the diff. If the description could substitute for reading the diff, it is too long.
> **Source:** `2026-06-05_pr-description-lessons.md`
> **Why:** Over-verbose descriptions duplicate the diff and go stale immediately after a fix commit.

> **Rule:** The Testing Endpoints section must include the side-effect or auto-transition case — it is more valuable than the happy-path example.
> **Source:** `2026-06-05_pr-description-lessons.md`
> **Why:** Reviewers and QA need to verify the non-obvious paths, not just the nominal case.

> **Rule:** Never add new columns to a base migration (e.g. `0001_01_01_000000_create_users_table.php`); always create a new timestamped migration instead.
> **Source:** `PR-reviews/2026-06/BE/2026-06-25_pr-review-sym-197.md`
> **Why:** Any environment where the table already exists (staging, prod, other dev machines) will not re-run a migration already recorded — the new columns will be silently missing.

> **Rule:** Seeded lookup/catalog data inside a migration follows the SAME no-edit-after-run discipline as schema. A migration does not re-run where its table already exists, so adding a value to a seeded catalog (e.g. a new `Role::case` materialized into `roles` by `2026_07_14_000001_create_roles_table.php`) requires a NEW follow-up migration with its own idempotent `updateOrInsert` for just that value — never edit the original seed loop. Do not build a re-runnable seeder for this; document the convention in the migration itself.
> **Source:** DEVSYM-376 implementation, 2026-07-14 (`sessions/2026-07/BE/DEVSYM-376/2026-07-14_19-25_internal-user-account-management.md`).
> **Why:** Same silent-drift failure mode as adding columns to an already-run migration, applied to seeded data instead of schema: environments where `roles` already exists (staging, prod, other dev machines) never re-run the seed loop, so a value added by editing it is silently missing everywhere the migration already ran.

> **Rule:** Add an inline comment in `.env.example` for any intentionally empty env var — the file is often the first setup document a dev reads.
> **Source:** `PR-reviews/2026-06/BE/2026-06-25_pr-review-sym-197.md`
> **Why:** Unexplained empty values look like mistakes and cause misconfiguration.

> **Rule:** `personal_access_tokens.tokenable_id` is now `VARCHAR(255)` (widened from the Sanctum-default unsigned bigint in migration `2026_07_15_000001`), because `InternalUser` has a UUID PK. Any new Sanctum-token-issuing model may be bigint- OR UUID-keyed; the column holds both and Sanctum resolves the model via `tokenable_type`. Do NOT re-narrow it. A UUID tokenable against a bigint column fails Postgres `22P02`. The stack is Postgres-only, so the migration uses raw `DB::statement('ALTER ... USING ...')` — `->change()` cannot cast bigint→varchar without dbal (not installed), so prefer raw ALTER+USING for any future type change here.
> **Source:** `sessions/2026-07/BE/DEVSYM-415/2026-07-14_22-18_internal-user-authentication.md`
> **Why:** Verified against a `pg_dump` copy of dev with a real preexisting `User` token: the ALTER preserved the token row byte-identical (md5 unchanged) and unblocked UUID `InternalUser` tokens. Widening (not a second table) keeps one polymorphic tokens table for both identity models.
>
> **Rule (down-migration):** The rollback of any string→bigint (or narrowing) column change on a polymorphic table shared by mixed-key models MUST `DELETE` the incompatible-key rows BEFORE the `ALTER ... USING x::BIGINT`, or the cast aborts. For `personal_access_tokens`, `down()` deletes `WHERE tokenable_type = 'App\Models\InternalUser'` first (FQCN verified against `createToken()`; no morph map). This deliberately drops internal-user sessions on rollback — document that intent in-code so it reads as designed, not accidental. NOTE: that WHERE assumes the literal FQCN; if a `Relation::morphMap()`/`enforceMorphMap()` is ever introduced, `tokenable_type` becomes the alias, the DELETE silently matches nothing, and the rollback breaks again — update the string with any morph map.
> **Source:** DEVSYM-415 Opus PR review should-fix, 2026-07-14.
> **Why:** A UUID `tokenable_id` cannot cast to bigint (`22P02`), so a naive `down()` would fail for anyone who had issued an internal-user token — the rollback path must be exercised, not assumed reversible.

---

## SPACE time logging

_(No new rules this cycle — format already covered in the solve-ticket skill.)_

---

## Code quality

> **Rule:** When a loop is bounded by a defensive cap (max pages, max records, max retries) rather than its natural terminating condition, log at the branch where the cap fires (resource + id + the limit). No silent caps — a partial result returned because the "impossible" case happened must be diagnosable, not invisible.
> **Source:** `sessions/2026-07/BE/2026-07-13_correction-pr-review-374-findings.md` (DEVSYM-374 PR #141, finding 2)
> **Why:** `PortfolioCacheService::fetchAllPages()` stopped at `MAX_PAGES` (50 pages / 50k rows) with no log; if the cap ever ends the walk it means a genuinely huge PM or a garbage `total` — exactly the case you need surfaced, not swallowed.

> **Rule:** In any feature test that accesses authenticated user properties (`email`, `id`, etc.) OR asserts a guard that discriminates by user identity/type, add `$this->withoutMiddleware(\App\Http\Middleware\BypassAuthForDev::class)` — `actingAs()` alone is overridden by `BypassAuthForDev` when `DEV_AUTH_BYPASS=true` and no bearer token is present. Note it fires on `app()->environment('local')`, and the test suite runs as `local` here, so this bites test runs, not just manual Docker use.
> **Source:** `2026-06-08_correction-feature-test-auth.md`; extended DEVSYM-415 (`sessions/2026-07/BE/DEVSYM-415/2026-07-14_22-18_internal-user-authentication.md`).
> **Why:** The middleware replaces the `actingAs()` user with the RM admin `User` (looked up by `rm_user_id = config('rent_manager.admin_user_id')`). Beyond null-property 500s, this silently swaps the identity **type**: a guard checking `instanceof InternalUser` saw the injected `User` and returned 403 for an `actingAs($internalUser, 'sanctum')` test until the bypass was disabled. Old `rm_super_admin` tests passed only because the injected admin happened to satisfy that guard.

> **Rule:** Always use `User::factory()->create()` (not `make()`) in feature tests — guards that do DB lookups will not find a non-persisted model.
> **Source:** `2026-06-08_correction-feature-test-auth.md`
> **Why:** `make()` produces an in-memory object with no DB row; any middleware or guard that queries the database will fail to find it.

> **Rule:** `DEV_AUTH_BYPASS=true` in the Docker environment wins over `phpunit.xml` env tags — any test that expects a 401 will pass as 200 in this environment.
> **Source:** `2026-06-05_pr-description-lessons.md`
> **Why:** The bypass middleware fires before Sanctum; phpunit env overrides cannot suppress it unless the middleware is explicitly excluded.

> **Rule:** Every new or modified endpoint must have a feature test asserting the response shape; this is a hard requirement, not a suggestion.
> **Source:** `PR-reviews/2026-06-19_pr-review-sym-329.md`
> **Why:** Without a test, shape regressions (changed envelope, missing fields) go undetected until FE breaks.

> **Rule:** Mock typed constructor/method dependencies with PHPUnit's `$this->createMock(Class::class)` (returns `Class&MockObject`), NOT `Mockery::mock()`. Use `->expects($this->exactly(n))`/`->method()`/`->with()`/`->willReturn*()` for expectations.
> **Source:** `2026-07-10_18-40_correction-phpstan-mockery-vs-createmock.md`
> **Why:** Mockery mocks are untyped, so PHPStan level 5 (run by the captainhook pre-commit hook) raises `argument.type` on the constructor and `method.notFound` on `->once()`/`->twice()`; the commit is blocked or a new baseline entry is forced. `createMock` is typed against the class and the repo's dominant convention.

> **Rule:** There is no refresh token in this codebase — `POST /api/auth/login` is the only way to obtain a new token; do not add refresh logic without an explicit ticket.
> **Source:** `2026-06-10_13-15_devsym-305-auth-token-ttl.md`
> **Why:** Sanctum personal access tokens are not refreshable by design; a `POST /api/auth/refresh` implementation is a deliberate future decision, not an oversight.

> **Rule:** Any credential-checking login must run a bcrypt comparison (`Hash::check`) on EVERY branch — including unknown email and password-never-set — using a constant valid-bcrypt `DUMMY_HASH` fallback when there is no real hash. Never let a `||` short-circuit return before the hash on a missing account. (Assign `$user?->password` to a variable first, then `?? DUMMY_HASH` — `$user?->password ?? DUMMY` on one line trips Larastan `nullsafe.neverNull`.)
> **Source:** `sessions/2026-07/BE/2026-07-14_22-18_correction-timing-safe-login.md` (DEVSYM-415)
> **Why:** A fast-fail (~3 ms) on a missing account vs a real bcrypt check (~270 ms) lets an attacker enumerate which accounts exist by response timing. Measured after fix: success 265 ms vs all failures 270–274 ms — indistinguishable.

> **Rule (mechanism of the above):** `nullsafe.neverNull` fires on the **syntactic position**, NOT on whether the receiver is nullable. Any `?->` whose result is the left operand of `??` is reported — verbatim: `Using nullsafe property access "?->prop" on left side of ?? is unnecessary. Use -> instead.` — because `??` absorbs the null either way. It fires even when the receiver is genuinely `?Foo`. Two valid remedies: assign the property to a variable first, then `??` (DEVSYM-415), or drop the `??` entirely and let `(int) null === 0` / the downstream guard handle it (DEVSYM-517 `TaskService::pending()`).
> **Source:** DEVSYM-517 PR #313 review nit, empirically re-run 2026-07-31 (`sessions/2026-07/BE/2026-07-31_18-04_correction-pr-review-517-findings.md`).
> **Why:** A reviewer read the DEVSYM-415 entry as being about non-null narrowing and concluded the rule couldn't apply to a nullable receiver, so the guard comment looked unfounded. It applies. Naming the real mechanism stops that re-derivation. Corollary: when a review finding is self-flagged as unverified ("could not run PHPStan"), run the tool — don't defer to the reviewer and don't defend the original wording.

---

## FE-specific

> **Rule:** Use `refetchQueries` (not `invalidateQueries`) when you need guaranteed immediate data sync after a mutation — `invalidateQueries` is deferred by the global `staleTime: 1000 * 60` and will not trigger a network request until the stale window expires.
> **Source:** `2026-06-16_devsym-346-daily-planner-auto-refresh.md`
> **Why:** After a delete/mutation, `invalidateQueries` showed a ~2-minute delay before the Daily Planner updated because the stale window hadn't expired.

> **Rule:** Always check the query key factory before assuming a task mutation covers dashboard queries — `dashboardKeys.all = ['dashboard']` and `taskKeys.all = ['tasks']` are separate namespaces.
> **Source:** `2026-06-16_devsym-346-daily-planner-auto-refresh.md`
> **Why:** `invalidateAllTaskQueries` does not touch `['dashboard']`; mutations that affect the Daily Planner must explicitly invalidate or refetch under `dashboardKeys`.

> **Rule:** Never use conflicting Tailwind utilities that target the same CSS property at the same breakpoint (e.g. `sm:truncate` + `sm:text-clip` both set `text-overflow`); winner depends on Tailwind class-generation order, which is non-deterministic from source.
> **Source:** `PR-reviews/2026-06/FE/2026-06-24_14-30_pr-review-devsym-347.md`
> **Why:** One of the two utilities silently wins and the other is ignored — behavior changes across Tailwind versions.

> **Rule:** Use TanStack Router's type-safe matching (`useMatch({ from: '/settings', shouldThrow: false })`) instead of `pathname.startsWith('/settings')` for route-conditional UI logic.
> **Source:** `PR-reviews/2026-06/FE/2026-06-24_14-30_pr-review-devsym-347.md`
> **Why:** Hardcoded string checks produce silent regressions when the route path is renamed; typed matching gives a compile error.

> **Rule:** Document the stacking context scale (e.g. `backdrop=10050, modal=10060, toast=10100`) in a shared const or comment block whenever magic z-index values span multiple files.
> **Source:** `PR-reviews/2026-06/FE/2026-06-24_17-00_pr-review-devsym-350.md`
> **Why:** Scattered magic z-index values with no single source of truth require manual coordination across every future change and are easy to break silently.

> **Rule:** Never apply `pointer-events-none` to a scroll container that wraps interactive content — the scrollbar thumb becomes non-interactive and touch scrolling outside the inner card breaks.
> **Source:** `PR-reviews/2026-06/FE/2026-06-24_17-00_pr-review-devsym-350.md`
> **Why:** `pointer-events-none` on the scroll wrapper kills scrollbar drag and touch-initiated scroll that starts outside the `pointer-events-auto` child; use a separate transparent backdrop element for click-through instead.

---

## Architecture — active decisions

> **Rule (discovery):** The Voice Caller (`app/Modules/VoiceAI/`) is a turn-based phone IVR over Twilio `<Gather>`/`<Say>` — there is no ElevenLabs, no WebSocket, no media streaming. Do not assume or re-introduce streaming architecture.
> **Source:** `2026-06-23_19-28_voice-caller-architecture-discovery.md`
> **Why:** The project brief assumed ElevenLabs + WebSocket streaming; the actual code is entirely different. V3 task #8 (`VoiceAIWebSocketHandler`) contradicts the current architecture and must be re-scoped before estimating.

> **Rule:** Sanctum token TTL is now 30 days (`60 * 24 * 30` in `config/sanctum.php`); `SANCTUM_TOKEN_EXPIRY` env var overrides it per environment.
> **Source:** `2026-06-10_13-15_devsym-305-auth-token-ttl.md`
> **Why:** Increased from 7 days as a stopgap while a proper refresh endpoint is planned; longer TTL = longer blast radius for a leaked token.

> **Rule:** Before wiring any real caller to `PmConnectionContext` (e.g. under DEVSYM-373/374), re-examine `EnsureRmTokenValid`'s expiry gate — it was validated only against the `UserSessionContext` path.
> **Source:** `sessions/2026-07/SymAssist-Backend/DEVSYM-404/space-time-log.md`
> **Why:** The `rm_password` allow-through added in DEVSYM-404 is safe only because every gated route resolves through `UserSessionContext` today; a route resolving through `PmConnectionContext` has a different self-heal path never checked against this gate.
> **RESOLVED (DEVSYM-374):** No change needed. DEVSYM-374's `/api/superuser/portfolio` route uses `['api','auth:sanctum','rm_super_admin']` — no `rm_token.valid` at all (same "no RM call on the caller's own session" precedent as DEVSYM-376). `EnsureRmTokenValid` only ever gates the calling superuser's own Sanctum+RM session; the actual RM calls ride per-`PmConnection` tokens via `PmConnectionContext`, which self-authenticates independently. The two are orthogonal — confirmed, not a latent bug.

> **Rule:** There are TWO identity models. `users`/`App\Models\User` is RM-coupled (rm_user_id, rm_token, rm_roles, linked_owner/tenant/vendor; login upserts by rm_user_id; `user_type` derived from RM roles) = external portal identity (owner/tenant/vendor) authenticated via RM. Any SymAssist-owned identity that is decoupled from RM (internal staff / superusers) belongs in a SEPARATE table (`internal_users`), never in `users`.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-376-be-internal-user-account-management.md`
> **Why:** Extending `users` for RM-decoupled staff would drag RM fields and RM-login semantics onto an identity the ticket mandates be RM-independent, breaking the decoupling.

> **Rule:** `rm_super_admin` (`EnsureIsSuperAdmin`) is the ONLY admin gate — it compares `user()->rm_user_id` to `config('rent_manager.admin_user_id')` (default 393, `RM_ADMIN_USER_ID`). There is NO local role/permission table behind admin authz. Reuse this middleware for any "admin-guarded" endpoint; do not invent a roles-based admin check.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-376-be-internal-user-account-management.md`
> **Why:** Building a parallel roles-based admin gate duplicates authz and drifts into DEVSYM-373's scope (role semantics/policy live there, not in feature tickets).

> **Rule:** There is NO local `roles`/permissions table. The `Users` module's `RoleController`/`RoleService` are RM remote-API passthroughs (`rentManagerService->getRoles/createRole`), NOT a local model — do not confuse them with a local roles table, and do not route local role data through RM.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-376-be-internal-user-account-management.md`
> **Why:** Wiring a local feature's roles through the RM passthrough would make an RM call where none is wanted and couple SA identity to RM.

> **Rule:** There is NO mail infrastructure in the backend — zero Mailables/Notifications, no `app/Mail`, no email views; `config/mail.php` is stock with `MAIL_MAILER` defaulting to `log`. A ticket saying "reuse the existing mailer" means: build on stock Laravel `Mail`/`Notification` (the first one in the app), and expect emails to land in the log until a real transport (SMTP/SES) is configured in env.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-376-be-internal-user-account-management.md`
> **Why:** Assuming a mailer service exists leads to a phantom-dependency dead-end; and shipping mail without flagging the unconfigured transport looks "done" but silently logs instead of sending.

> **Rule:** For enum/policy/state-machine-style domain machinery (e.g. DEVSYM-373's `Role` + policy + fan-out), mirror `app/Modules/Tasks/StateMachine/` (native backed enum + small stateless services, RM mapping kept OUT of the enum) — NOT the `Users/` CRUD skeleton (Controller/Service/Request/DTO). The ONLY SA-native enum today is `Tasks/StateMachine/TaskStatus`; `Role` is greenfield.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-373-role-policy-fanout-recon.md`
> **Why:** Pure domain machinery has no HTTP/CRUD surface; the CRUD skeleton would drag controllers/requests/routes onto logic that 373 mandates be table-less and endpoint-less.

---

## DEVSYM-399 foundation — as-built

> **Rule:** `pm_connections` (migration `2026_07_01_000001`) columns: `id`; `rm_location_id` (unsigned bigint, unique, NO FK); `pm_name` (nullable); `rm_credential_ref` (ENCRYPTED text — stores a `{username,password}` JSON blob, decrypted in `PmConnectionContext`); `status` (enum `pending|active|suspended`, default `pending`); `onboarded_at`; timestamps. There IS a credential column (`rm_credential_ref`) — the "no credential column" assumption is FALSE. "active" is a `status` enum value, not a boolean. The model has no relations/scopes.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-374-cross-pm-data-aggregation-and-caching-layer.md`
> **Why:** Refinements/code that assume no stored credentials, a boolean `active`, or a FK/relation on this table are wrong on all three counts; "all active PM connections a role reaches" must map to `status = 'active'` and a mapping that does not exist yet.

> **Rule:** DEVSYM-399 ships TWO `RentManagerContext` implementations: `UserSessionContext` (request-bound, reads `Auth::user()`, `locationId()` null) and `PmConnectionContext` (request-INDEPENDENT by construction, `locationId()` = `rm_location_id`, token via `credential_mode`). "Partner token" is a `credentialMode()` BRANCH inside `PmConnectionContext`, NOT a separate `PartnerTokenContext` class. `RentManagerContextResolver` exposes `clientForConnection`/`contextForConnection`/`clientForCurrentSession` but has ZERO callers (dead scaffolding). Default container binding is `RentManagerContext → UserSessionContext` (`AppServiceProvider`). `EnsureRmTokenValid` is request-only and applied to no route.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-374-cross-pm-data-aggregation-and-caching-layer.md`
> **Why:** A background/off-request Location-scoped client is buildable on the `PmConnectionContext` primitive, but the intended glue (`RentManagerContextResolver`) is unwired — treating it as "already wired" is drift; and there is no `PartnerTokenContext` class to reference.

> **Rule:** `pm_connections` holds one row per (corpid, location) — see `candidatesForCorpid()`/`firstForCorpid()`, which exist because a corpid can have several granted locations. So any value that is semantically **per-corpid** but stored on this table (`sa_*_role_id`, `sa_ptr_rm_user_id`) must be written to EVERY row of that corpid, not just one. Whoever writes it — a provisioning command or a human via tinker — must iterate `candidatesForCorpid()`. Never describe such a column as "per-corpid" in a docblock without saying it has to be replicated per row.
> **Source:** DEVSYM-517 PR #313 review nit, 2026-07-31 (`sessions/2026-07/BE/2026-07-31_18-04_correction-pr-review-517-findings.md`).
> **Why:** A half-provisioned corpid fails on ONE location only: a superuser acting for the un-provisioned location gets a silently empty result while the sibling location works fine. That asymmetry reads like a data problem in RM, not a missing column value, and is expensive to debug.

> **Rule:** Nothing in the app sets `pm_connections.status = 'active'` yet — only the migration default (`pending`) and tests write `status`; the Onboarding module never touches `PmConnection`, and there is no seeder/factory/endpoint that activates a connection. So 373's "onboarding hook" is FREE: a live `where status = 'active'` scope auto-includes any future activated PM with nothing to wire.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-373-role-policy-fanout-recon.md`
> **Why:** Treating activation as work 373 must build is wrong scope — activation belongs to onboarding/376; 373 only queries the live status.

---

## Caching layer — as-built

> **Rule:** The existing cache layer (DEVSYM-329/400) is SA-domain only; the Integrations layer caches ONLY the RM token. Two patterns: SWR via `Cache::flexible($key, [fresh, stale], $cb)` (`PropertyPerformanceService` `[300,900]`, `VacancyService`) and `Cache::remember` + version-counter (`TaskService` 5min + `TaskCacheVersion`, `TaskStatsService` 15min, `DailyPlannerService` 5min). Redis store, NO `Cache::tags()` (deliberate — not store-portable). All domain keys are `Auth::id()`-based. NO scheduled/proactive refresh anywhere — refresh is lazy TTL, write-triggered version bump, or `?refresh=1`. Domain TTLs are hardcoded class constants; only the token TTL is config-driven (`RM_TOKEN_CACHE_TTL_MINUTES`, default 10).
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-374-cross-pm-data-aggregation-and-caching-layer.md`
> **Why:** Reuse over duplication — a new caching feature extends this precedent (Location-keyed `Cache::flexible`), it does not invent a mechanism; and there is no existing per-connection key or proactive refresh to lean on.

> **Rule:** ZERO scheduled tasks (`routes/console.php` has only `inspire`) and ZERO queued jobs (`Integrations/Jobs` is `.gitkeep`; no `ShouldQueue`/`dispatch` repo-wide). Horizon `^5.43` + Redis queue infra is provisioned but idle — any refresh job would be first-of-its-kind.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-374-cross-pm-data-aggregation-and-caching-layer.md`
> **Why:** A scheduled cache-warm path is not a small add — it introduces the first off-request RM caller AND the first scheduled job; scope it as a deliberate boundary, not an incidental detail.

---

## Superuser stack ownership

> **Rule:** DEVSYM-373 owns cross-PM fan-out + role-reach policy; DEVSYM-374 owns caching + counts + PM-tagging on top. This supersedes any earlier note that 374 owns the fan-out or is "basic fan-out, no caching". (The `EnsureIsSuperAdmin` gate is documented above; note additionally that it is NOT connection-aware.)
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-374-cross-pm-data-aggregation-and-caching-layer.md`
> **Why:** 374 cannot assume the fan-out exists (373 is unstarted) and must not re-scope 373's policy; conflating the two produces an undefined 373→374 input contract.

> **Rule:** Final Superuser-stack ownership split. **DEVSYM-399** = credentials + `RentManagerContextResolver::clientForConnection`. **DEVSYM-373** = `enum Role` (SA source of truth) + an `active` scope on `PmConnection` + `SuperuserAccessPolicy::reachableConnections(Role)` + `CrossPmFanout::fanOut(Role, callable)` — pure stateless domain machinery, NO table/endpoint/auth. **DEVSYM-376** = `roles` table (seeded from `Role::cases()`) + `internal_users` + `role_id`. **DEVSYM-374** = caching + counts + PM-tagging + HTTP endpoint, wrapping 373's `fanOut`. Auth/login = a later ticket. "Assignable role" is resolved: 373 DEFINES roles, 376 ASSIGNS them.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-373-superuser-roles-and-cross-pm-access-policy.md`
> **Why:** Prevents the recurring 373/374/376 scope bleed — role *semantics* live in 373, the role *store* in 376, and *consumption* in 374; each ticket builds only its layer.

> **Rule:** 373→374 contract: `CrossPmFanout::fanOut` returns `array<int rm_location_id, array>` — one raw decoded-JSON array per reached connection (`RentManagerHttpClient::get()` returns `array`). 374 supplies the per-connection operation (the `callable`), caches per `rm_location_id`, then merges/tags/counts. NEVER pre-merge in 373 — the per-Location split is exactly what makes 374's per-Location caching possible.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-373-superuser-roles-and-cross-pm-access-policy.md`
> **Why:** A pre-merged blob collapses the Location keys 374 needs to cache/tag on, forcing 374 to re-split or abandon per-Location caching.

> **Rule:** RM rate limits are isolated per customer DB (DEVSYM-392), so a single fan-out pass (one request per Location) is well within budget. Do NOT stagger the fan-out and do NOT reach for `RentManagerHttpClient::poolGet`/`pool` concurrency for it (YAGNI). Supersedes the earlier "stagger the fan-out (500 req/60s)" note.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-373-superuser-roles-and-cross-pm-access-policy.md`
> **Why:** The 500 req/60s cap is per user+DB, not global; fanning out across N Locations issues one request per DB, so staggering/pooling adds complexity for a limit that is never in play.

> **Rule:** Any aggregation built on `CrossPmFanout` must surface which connections were reached vs skipped alongside its counts (DEVSYM-374's `PortfolioCacheService` exposes `meta.connections.{reachable,reached,skipped}`, derived from `SuperuserAccessPolicy::reachableConnections()` vs the fan-out's returned keys). The fan-out's per-item isolation drops broken connections silently (logs server-side only), so counts alone cannot distinguish "PM empty" from "PM dropped". Derive `reached` from the fan-out KEYS, not the row count — a reached-but-empty PM has zero rows but is still reached.
> **Source:** `sessions/2026-07/BE/2026-07-13_correction-pr-review-374-findings.md` (DEVSYM-374 PR #141, finding 1)
> **Why:** Silent under-reporting in a portfolio view whose whole purpose is cross-PM completeness is a correctness bug, not a cosmetic one — a superuser reading the dashboard would trust a number that dropped a whole PM.

---

## Process gates

> **Rule:** For tickets depending on already-merged code, run a read-only repo-diagnosis (report findings, no plan/code) BEFORE refining or coding.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-374-cross-pm-data-aggregation-and-caching-layer.md`
> **Why:** Silent assumptions about repo shape are the #1 source of drift — in this session two invented "architecture decisions" collapsed once the existing cache + off-request-context reality was surfaced.


> **Rule:** On Frontier-classified tickets, Claudio's initial plan tends to surface only the traps documented in the *current* ticket's own body — it does not by default cross-check RULES.md or sibling tickets in the same stack for traps that apply here too. Before approving a Frontier plan, explicitly diff it against: (a) this ticket's own "Known traps" section, (b) sibling tickets in the same stack (e.g. 373/374/376 for Superuser) for shared traps, (c) the relevant RULES.md sections for the module being touched.
> **Source:** DEVSYM-374 plan review, 2026-07-13 (chat session — file under `sessions/2026-07/BE/DEVSYM-374/` if this pattern repeats).
> **Why:** In the DEVSYM-374 plan-gate review, three real gaps surfaced across two review rounds — `rm_token.valid` left in the middleware despite the DEVSYM-376 precedent for the same actor type; a missing stale/deleted-record filter despite it being in both the ticket's own traps *and* RULES.md's "RM Integration" section; a hardcoded `Role` placeholder with no forward-reference comment for when 376 lands. All three were caught only because the reviewer manually cross-referenced — the plan itself didn't surface them. Cheaper to check this explicitly at plan-gate time than in code review or production.
>
> **Rule:** RM does not document an `IsActive`/`IsDeleted`-equivalent field for Tenants or Owners (unlike Property, which has `IsActive`, and Units, which inherit it via embedded `Property.IsActive`). Do not invent one — verify with apisupport@rentmanager.com before assuming a filter exists for these two resources.
> **Source:** DEVSYM-374 implementation, 2026-07-13.
> **Why:** Confirmed absent during 374's field research; inventing a guessed field name either silently fails to filter (wrong name, no error) or breaks the request. If accurate counts become a real requirement, ask RM support directly rather than guessing again.

> **Rule:** Bash output in `symassist-backend` is redaction-mangled — identifiers like `role_id`, `active`, `Sanctum` are rewritten to `n`/`ln` in `grep`/`cat` output. Use the **Read tool** for exact signatures; treat shell output as unreliable for identifiers.
> **Source:** `refinement/BE/2026-07/2026-07-09-devsym-373-role-policy-fanout-recon.md`
> **Why:** Confirming a signature or column name from mangled shell output silently reads the wrong identifier; the Read tool is unfiltered.

---

## Open / unresolved

> **Owner:** Pending dev-lead / product confirmation.
> **Context:** RM `NoteType` for `createIssueNote()` audit notes (emergency bypass) — no explicit type confirmed; currently passing no type.
> **Source:** `2026-06-08_devsym-152-emergency-handling.md`

> **Owner:** Pending product decision.
> **Context:** `HumanReviewJob` in the VoiceAI self-learning loop is a stub (only `Log::warning`) — confirm whether a real review channel (email/Slack/UI) is needed before relying on the confidence-decay loop in production.
> **Source:** `2026-06-23_19-28_voice-caller-architecture-discovery.md`

> **Owner:** Pending V3 scoping session.
> **Context:** pgvector tables (`voice_ai_tool_patterns`, `rrm_api_methods`) have no `voice_client_id` column — if patterns must be isolated per PM in V3, a schema migration is needed that is not currently listed in V3.0 tasks.
> **Source:** `2026-06-23_19-28_voice-caller-architecture-discovery.md`

> **Owner:** Pending product decision.
> **Context:** V3 task #8 (`VoiceAIWebSocketHandler`) assumes a streaming/WebSocket architecture that does not exist in v2; the task is likely a leftover from an earlier design and should be re-scoped before estimating.
> **Source:** `2026-06-23_19-28_voice-caller-architecture-discovery.md`

> **Owner:** Pending product decision.
> **Context:** `VOICEAI_DYNAMIC_DISCOVERY` flag gates the V2 RAG path — confirm whether it is intended to ship ON in production or stay permanently behind the flag.
> **Source:** `2026-06-23_19-28_voice-caller-architecture-discovery.md`

> **Owner:** Pending poolVacancy follow-up PR.
> **Context:** `poolVacancy` in `RentManagerService` does not apply RM mappers inline; raw RM rows leak into `VacancyService`. Deferred from DEVSYM-329 review as a follow-up PR.
> **Source:** `PR-reviews/2026-06-19_pr-review-sym-329.md`

> **Owner:** DEVSYM-415 (login + guard for internal_users).
> **Context:** `PortfolioCacheService::CALLER_ROLE_PLACEHOLDER` (DEVSYM-374) is hardcoded to `Role::PortfolioManager` since there is no real source yet for "which role is the authenticated superuser acting as" — `SuperuserAccessPolicy::reachableConnections()` ignores `$role` today, so this is inert, but must be replaced with the real caller's role once DEVSYM-415 resolves auth.
> **Source:** `sessions/2026-07/BE/DEVSYM-374/2026-07-13_cross-pm-caching-layer.md`
> **RESOLVED (DEVSYM-415, merged 2026-07-15):** login + guard shipped. The placeholder still needs replacing with the authenticated caller's real role in a future per-role-gating ticket, but the auth path it was waiting on now exists.


> **Owner:** DEVSYM-415 (login + guard for internal_users).
> **Context:** `EnsureIsSuperAdmin` (aliased as `rm_super_admin`, gates DEVSYM-374's Portfolio endpoint) checks `$request->user()->rm_user_id` against a single hardcoded `config('rent_manager.admin_user_id')` — it authenticates against the RM-coupled `users` table. DEVSYM-376's `internal_users` have no `rm_user_id` and no RM login, so once 376 ships, no internal_user can pass this gate. DEVSYM-415 covers replacing/extending this guard.
> **Source:** DEVSYM-374 PR review, 2026-07-13.
> **RESOLVED (DEVSYM-415, merged 2026-07-15):** `EnsureIsInternalUser` (alias `internal_user`) now gates the Portfolio route; `rm_super_admin` retained only for the internal-user CRUD (documented coexistence).

PHPUnit `<env>` tags do not override an OS environment variable that is already set unless `force="true"`. `docker-compose.yml` sets `APP_ENV: local` on the `app` container, so any unforced `<env>` in `phpunit.xml` loses silently and the suite runs in the wrong environment. When adding or auditing test env vars: force them explicitly, and verify with `app()->environment()` inside a running test rather than inferring from test results. A cached config file also defeats the force — once `config:cache` has run, `env()` returns null and the forced value never reaches `config('app.env')`.


Never use `git checkout --` or `git reset --hard` as an undo mechanism on a file that holds uncommitted work — it reverts everything uncommitted in that file, not just the change you meant to discard. Use a `cp` backup and restore from it. This has now caused damage twice: a review session that ran `reset --hard` and destroyed the working tree, and a session that silently reverted three of four pending fixes while undoing a test mutation. The second nearly shipped as a complete-looking commit.

A mutation test must assert that the mutation actually applied to the file before running the test. A replacement that silently fails to match produces a false green: you conclude the assertion doesn't catch the defect when in fact you never introduced it. Verify the text changed, then run.