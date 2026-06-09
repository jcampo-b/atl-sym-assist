# Skill: Git Workflow

## When to read this
Before any git operation (branch creation, commit, PR).

## Branch strategy
- `dev` is the integration branch — it always holds the latest stable state.
- ALWAYS branch from `dev`, never from another feature branch.

```bash
git checkout dev
git pull origin dev
git checkout -b feat/short-description
```

- Branch types: `feat/`, `fix/`, `refactor/`, `docs/`, `chore/`.
- Never branch from `feat/*` directly.

## Commit conventions (Conventional Commits)
- Format: `type(scope): short description`
- Types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`.
- Scope: module name or area (`tasks`, `integrations`, `onboarding`, …).
- Examples:
  - `feat(tasks): add schedule[] filter with overdue/pending buckets`
  - `fix(integrations): always embed Status on ServiceManagerIssue list`
  - `refactor(tasks): extract buildInClause to mapper`

## PR requirements
- **Title** — `Type[TICKET]: Layer - brief description`
  - `Type`: `Feat`, `Fix`, `Hotfix`, `Refactor`, `Docs`, `Test`, `Chore`, `Perf`
  - `Layer`: `BE`, `FE`, or omit if full-stack
  - Examples:
    - `Feat[DEVSYM-319]: BE - status transition automation engine`
    - `Fix[DEVSYM-184]: BE - exception auth roles`
    - `Refactor[DEVSYM-201]: FE - task card component`
- **Description** — one or two sentences: what and why.
- **Ticket** — `[Ticket #XXX](link)`.
- **Type of Change** — check one.
- **Changes Made** — 4-5 bullets maximum. No implementation details —
  those are in the diff. Only what changed and why it matters.
- **Testing** — checkboxes only.
- **Checklist** — checkboxes only.
- **Screenshots / Examples** — only when a visual aid genuinely helps the reviewer. Omit if empty.
- **Additional Notes** — only for decisions, known gaps, or follow-ups not captured elsewhere. Omit if empty.

## PR body template
This is the canonical template. Fill every section; omit only those marked optional.

```markdown
## Description
Brief description of what this PR does and why it's needed.

## Ticket
- [Ticket #XXX](link-to-ticket) - Brief ticket description

## Type of Change
- [ ] Feature (`feat:`)
- [ ] Bug fix (`fix:`)
- [ ] Hotfix (`hotfix:`)
- [ ] Refactoring (`refactor:`)
- [ ] Documentation (`docs:`)
- [ ] Tests (`test:`)
- [ ] Chore (`chore:`)
- [ ] Performance (`perf:`)

## Changes Made
- 
- 
- 

## Testing
- [ ] Unit tests pass
- [ ] Feature tests pass
- [ ] Manual testing completed
- [ ] Tested in local environment
- [ ] API documentation updated (if applicable)

## Checklist
- [ ] Code follows project conventions and naming standards
- [ ] Tests added/updated for new functionality
- [ ] Documentation updated (if needed)
- [ ] No breaking changes (or documented if needed)
- [ ] Code is self-documenting and readable
- [ ] Dependencies updated (if applicable)
- [ ] Migration files created/updated (if applicable)

## Screenshots / Examples (if applicable)
<!-- Add screenshots, API examples, or other visual aids if relevant -->

## Additional Notes
<!-- Any additional information, context, or concerns -->
```

## Opening a PR — step by step

### 1. Switch to the Braintly account
# Braintly account = jcampo-b (NOT jhonattancampo — that is the personal account)
```bash
gh auth switch -u jcampo-b
gh auth status  # verify jcampo-b is active
```

### 2. Open the PR with an empty body
```bash
gh pr create \
  --base dev \
  --head <branch> \
  --title "<title>" \
  --body ""
```

### 3. Generate the PR body from the template
Read `.atl/pull_request_template.md` and fill it with the ticket's specific
content. Write the result to `/tmp/pr-body.md` using the file creation tool —
NOT a bash heredoc (heredocs cannot contain triple backticks without breaking
the shell parser).

### 4. Present the filled body to the user
Call present_files on `/tmp/pr-body.md` so the user can copy and paste it
directly into the GitHub PR description before merging.

## Testing Endpoints — the most important section
Add this section to every PR that touches any API surface. It lets reviewers
copy-paste each request and verify behavior in under a minute without reading
the code. Always include the side-effect or auto-transition case — it is more
valuable than the happy path.

Reference format:

### 1. Valid transition
```
PATCH http://127.0.0.1:8000/api/tasks/214925/status
Content-Type: application/json
Authorization: Bearer {{token}}

{ "status": "approval" }
```
```json
{
  "data": {
    "id": 214925,
    "status_id": 52
  }
}
```

### 2. Invalid transition — skip states
```
PATCH http://127.0.0.1:8000/api/tasks/214925/status
Content-Type: application/json
Authorization: Bearer {{token}}

{ "status": "completed" }
```
```json
{ "message": "Cannot transition from approval to completed." }
```
Expected: 422

### 3. Side-effect / auto-transition case
Requires: task in dispatch (status_id=53) with all subtasks completed=true
```
PATCH http://127.0.0.1:8000/api/tasks/214925/status
Content-Type: application/json
Authorization: Bearer {{token}}

{ "status": "production" }
```
Expected: 200. Verify with `GET /api/tasks/214925` after — `status_id` should be `55` (review), not `54` (production).

Rules:
- Use real IDs and values where possible.
- Include the expected response body, not just the status code.
- Add `Requires:` above the request block if a specific data state is needed.
- For side-effect cases, tell the reviewer exactly what to verify after.
- Omit only for PRs with zero API surface (pure refactor, docs, config).

## Before opening a PR
Sync with dev first:
```bash
git fetch origin
git merge origin/dev
```
If there are conflicts, resolve them and run `docker compose exec app php artisan test` before proceeding.
Confirm that any failing tests were already failing on `dev` — not introduced by this branch.

## Before merging
- `docker compose exec app php artisan test` must pass.
- `php artisan route:list` to verify no route conflicts.
- `php artisan l5-swagger:generate` if any OpenApi attributes changed.