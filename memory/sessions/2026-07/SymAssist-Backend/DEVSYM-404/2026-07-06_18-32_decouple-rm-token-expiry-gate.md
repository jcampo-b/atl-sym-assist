# DEVSYM-404 — Decouple RM token expiry gate from cache TTL in EnsureRmTokenValid

Branch: `fix/devsym-404-decouple-rm-token-expiry-gate-from-cache-ttl` (from `dev`)

## What was done
`EnsureRmTokenValid` returned a hard 401 the instant `rm_token_expires_at` was
in the past, cutting the request before `RentManagerHttpClient`'s existing
reauth-on-401 self-heal could run. With DEVSYM-400 (PR #128) shortening the
shared TTL to 10 min, that turns a rare re-login into a recurring 401 storm.
Fix: the gate now hard-blocks only users with no self-heal path — added
`&& ! $user->rm_password` to the 401 condition. Users with `rm_password` pass
through; `UserSessionContext` re-derives a fresh token downstream.

TDD per ticket: committed a characterization test pinning current behavior
(expired + rm_password → 401) first, then flipped that case to allow-through
(200) in the fix commit alongside the middleware change.

## Files touched
- `app/Http/Middleware/EnsureRmTokenValid.php` — added `! $user->rm_password` to the 401 guard + WHY comment.
- `tests/Feature/Http/Middleware/EnsureRmTokenValidTest.php` — new feature test (allow-through with password, preserved 401 without, valid-token pass-through).

## Decisions made
- Minimal conditional, no new interface/abstraction (ticket anti-pattern).
- No changes to `config/rent_manager.php`, `UserSessionContext`, `PmConnectionContext`, or context resolution (out of scope).
- Test uses `withoutMiddleware(BypassAuthForDev::class)` + `User::factory()->create()` (RULES.md) so the built user isn't swapped and props persist.
- Blast-radius verified empirically (see `space-time-log.md`): every `rm_token.valid`-gated route resolves through `UserSessionContext`; `PmConnectionContext` has zero real callers today.

## Verification
- New middleware tests: 3 passed.
- Full suite: 22 failed / 2 risky / 228 passed on branch vs 22 failed / 2 risky / 225 passed on `dev`. Same 22 pre-existing failures (DEV_AUTH_BYPASS 401→200, TaskComment LogicException, redis counters, catalogs). Zero regressions; +3 passing tests.
- pint + phpstan clean on both changed files (run inside Docker).

## Open questions
- Forward-looking flag promoted to RULES.md: before wiring a real `PmConnectionContext` caller (DEVSYM-373/374), re-examine this gate.
- SPACE dev-provided times still pending from Johnny (see `space-time-log.md`).
