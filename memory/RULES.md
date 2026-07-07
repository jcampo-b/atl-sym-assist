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

> **Rule:** ivfflat pgvector indexes built on empty tables yield 0 centroids and every similarity search returns 0 rows; run sequential scans (`seq-scan`) until the table reaches 5,000+ rows, then add `ivfflat` with `lists = sqrt(n)`.
> **Source:** `2026-06-23_19-28_voice-caller-architecture-discovery.md`
> **Why:** ivfflat centroids are computed at index-creation time; empty → 0 centroids → 0 results always.

---

## Layer discipline

> **Rule:** `poolVacancy` RM service specs must include a `map` callable applying the appropriate `RentManagerVacancyMapper` method — raw RM PascalCase rows must never reach the SA `VacancyService`. [verify-duplicate]
> **Source:** `PR-reviews/2026-06-19_pr-review-sym-329.md`
> **Why:** `poolDashboard` already follows this pattern correctly; inconsistency caused the SA layer to call RM mapper methods directly, violating the cardinal layer rule.

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

> **Rule:** Add an inline comment in `.env.example` for any intentionally empty env var — the file is often the first setup document a dev reads.
> **Source:** `PR-reviews/2026-06/BE/2026-06-25_pr-review-sym-197.md`
> **Why:** Unexplained empty values look like mistakes and cause misconfiguration.

---

## SPACE time logging

_(No new rules this cycle — format already covered in the solve-ticket skill.)_

---

## Code quality

> **Rule:** In any feature test that accesses authenticated user properties (`email`, `id`, etc.), add `$this->withoutMiddleware(\App\Http\Middleware\BypassAuthForDev::class)` — `actingAs()` alone is overridden by `BypassAuthForDev` in Docker when `DEV_AUTH_BYPASS=true`.
> **Source:** `2026-06-08_correction-feature-test-auth.md`
> **Why:** The middleware replaces the `actingAs()` user with a DB-looked-up user who may have null properties, causing 500s in Docker even with correct test setup.

> **Rule:** Always use `User::factory()->create()` (not `make()`) in feature tests — guards that do DB lookups will not find a non-persisted model.
> **Source:** `2026-06-08_correction-feature-test-auth.md`
> **Why:** `make()` produces an in-memory object with no DB row; any middleware or guard that queries the database will fail to find it.

> **Rule:** `DEV_AUTH_BYPASS=true` in the Docker environment wins over `phpunit.xml` env tags — any test that expects a 401 will pass as 200 in this environment.
> **Source:** `2026-06-05_pr-description-lessons.md`
> **Why:** The bypass middleware fires before Sanctum; phpunit env overrides cannot suppress it unless the middleware is explicitly excluded.

> **Rule:** Every new or modified endpoint must have a feature test asserting the response shape; this is a hard requirement, not a suggestion.
> **Source:** `PR-reviews/2026-06-19_pr-review-sym-329.md`
> **Why:** Without a test, shape regressions (changed envelope, missing fields) go undetected until FE breaks.

> **Rule:** There is no refresh token in this codebase — `POST /api/auth/login` is the only way to obtain a new token; do not add refresh logic without an explicit ticket.
> **Source:** `2026-06-10_13-15_devsym-305-auth-token-ttl.md`
> **Why:** Sanctum personal access tokens are not refreshable by design; a `POST /api/auth/refresh` implementation is a deliberate future decision, not an oversight.

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
