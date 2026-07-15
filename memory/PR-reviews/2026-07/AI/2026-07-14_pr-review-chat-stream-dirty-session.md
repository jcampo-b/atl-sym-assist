## Review opened
- Timestamp: 2026-07-14
- Repo: AI
- Branch: fix/chat-stream-dirty-session
- Base branch: main (PR's actual base; repo default of `dev` did not apply here)
- Issue: chat-stream-dirty-session (no ticket ID in branch name)

## Dev's stated focus
PR #12 — "fix: stop LLMProviderError from leaving the DB session dirty". The
`/chat/messages` endpoint intermittently failed with `sqlalchemy.exc.PendingRollbackError`
after a timeout. Root cause: `_run_tool_loop` re-raised `LLMProviderError` immediately,
skipping the rollback + partial-message persistence every other failure gets — leaving
the session in rollback-required state. The failure only surfaced later, misleadingly,
when `get_db`'s teardown tried to commit the already-broken session, and the real
exception was never logged locally (only sent to Sentry).

Changes:
- Removes the `LLMProviderError` special case in `_run_tool_loop` so it goes through the
  same rollback/partial-persist path as any other exception.
- Fixes `_sse_generator`'s log call (`logger.error` with no detail → `logger.exception`)
  so the real cause is visible in CloudWatch.
- Splits `get_db`'s teardown-commit failure handling: `PendingRollbackError` logs as a
  warning (echo of an already-reported earlier error), any other commit failure still
  logs at exception level.
- Adds regression test `test_tool_loop_llm_provider_error_rolls_back_and_preserves_partial_message`.

## Files in scope
src/app/api/v1/chat.py
src/app/core/database.py
src/app/services/chat.py
tests/services/test_tool_loop.py

## Findings
None. Verified:
- `_user_facing_error` has no special case for `LLMProviderError` — falls to the same
  generic message it always did, so unifying the exception path doesn't change the
  user-facing message.
- `get_db`'s new `PendingRollbackError` branch is safe by construction: it can only fire
  when the request-handling code already caught+rolled-back+logged its own exception
  (true for both `_run_tool_loop` and `_run_text_only`), since any exception that
  propagates out of the generator hits the earlier `except Exception: rollback; raise`
  block instead.
- New test follows existing file conventions (inline `sqlalchemy.select`/`Message` import
  matches an existing test at line 131; no unnecessary `@respx.mock` since no backend
  call is made, consistent with `test_tool_loop_suggest_create_task_emits_frontend_action`).
- No dangling `LLMProviderError` references — still raised at the source in `llm.py`.

## Verdict
ready to merge

## SPACE log
- Time spent reviewing: 20 min
- Model: Sonnet 5
- Findings: 0 blockers, 0 should-fix, 0 nits
- Verdict: ready to merge
