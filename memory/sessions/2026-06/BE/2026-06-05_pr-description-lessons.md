# PR Description Lessons

## What was wrong
PR description was too verbose — included implementation details that belong
in the diff, not the PR comment.

## Rules to apply going forward
- Changes Made: 4-5 bullets max. No implementation details.
- Testing Endpoints: always include the side-effect/auto-transition case —
  it is more valuable than the happy path.
- If the PR description could replace reading the diff, it is too long.

## Other issues found this session
- RM assigns status ID 1 (<Unassigned>) to newly created issues — mapped
  to virgin on read path, writes still use ID 56.
- RM rejects ServiceManagerIssue updates without Title — always carry Title
  from the fetched raw issue.
- DEV_AUTH_BYPASS=true in Docker env wins over phpunit.xml env tags —
  401 tests will always pass as 200 in this environment.
- Auto-transition response shows pre-auto-transition status_id — FE should
  always GET after PATCH to get final state.
