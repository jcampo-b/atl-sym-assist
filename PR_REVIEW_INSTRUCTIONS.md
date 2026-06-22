# PR Review Instructions

> **Purpose.** This file defines the complete flow for a **review-only** pass on a
> branch. Paste it (or keep it loaded in the project) and open a chat with:
>
> > "Do a PR review. Repo: BE. Branch: `fix/devsym-319-status-edge-case`. The dev implemented: xxx"
>
> The agent then runs the whole flow below: read context → diff → review → record
> in memory → wait for approval → close. No code is written, committed, or pushed.

---

## Role

You are doing a **review-only** pass on a branch. You will NOT write code, commit,
push, or fix anything. Your output is a list of clear, actionable review comments
plus a memory record of the review.

## Workspace layout

Orchestration root:
`/Users/awesomejohnny/Development/Braintly/SymAssist`

Repos under it:
- Backend (BE): `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-Backend`
- Frontend (FE): `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-Frontend`
- AI Service: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-AI-Service`

Shared memory dir: `/Users/awesomejohnny/Development/Braintly/SymAssist/.atl/memory`

## Base branch per repo (what to diff against)

- BE → `dev`
- FE → `stage`
- AI Service → `dev`

If the user specifies a different base branch in chat, use that instead.

## What the user provides in their message

- The repo (BE / FE / AI Service).
- The branch to review (the one the dev pushed).
- A description of what the dev implemented / wants reviewed.

---

## Step 0 — Read context first

`cd` into the correct repo for the target. Then, before anything, read:
- `AGENTS.md`
- All files in `.atl/memory/`
- `linter.md`
- `.atl/skills/code-review/SKILL.md`

If the changes touch RM integration (BE), also read `docs/rent-manager-integration.md`
and `docs/rent-manager-filters.md`.

## Step 1 — Get the diff

```bash
cd <repo-path>
git fetch origin
git checkout <branch>
git diff origin/<base-branch>...<branch>
```

Use the base branch for that repo (BE=dev, FE=stage, AI=dev) unless the user
overrode it. Review every changed file in the diff.

## Step 2 — Review criteria

Evaluate the changes against, in this order:
1. The dev's stated focus (what they said they implemented / want reviewed).
2. `.atl/skills/code-review/SKILL.md`.
3. Layer discipline: RRM → RM → SA → FE. No PascalCase RM field names outside
   `app/Modules/Integrations/RentManager/`. FE/SA never coupled to RM.
4. Within-layer responsibilities (FormRequest = validation, Service = orchestration,
   StateMachine = transitions, Mapper = translation, Controller = thin).
5. Known RM traps in `AGENTS.md` / memory (Title required on PATCH, Status embed,
   StatusID 1 → Virgin, hard-delete, no OR in filters, read-only computed fields, etc.).
6. Linter rules in `linter.md` — flag anything that would break the linter.
7. Code quality rules: no over-engineering, no dead/defensive code, no speculative
   abstractions, no one-off methods extracted without a second caller.

(Criteria 3–6 are BE-specific. For FE and AI Service, apply the dev's focus, the
code-review skill, that repo's conventions, and its linter; skip RM-layer rules that
don't apply.)

## Step 3 — Output the review

Output ONLY review comments. For each finding:
- **File + line** (e.g. `app/Modules/Tasks/Services/TaskService.php:42`)
- **Severity**: `blocker` / `should-fix` / `nit`
- **What's wrong** — one line.
- **Why** — cite the rule, skill, trap, or doc it violates.
- **Suggested fix** — what to change (described, NOT applied).

Group findings by severity, blockers first. If a file is clean, don't mention it.
End with a one-line verdict: ready to merge / needs changes before merge.

## Step 4 — Record the review in memory

After producing the review, create a memory file in:
`/Users/awesomejohnny/Development/Braintly/SymAssist/.atl/memory/PR-reviews`

Create the `PR-reviews/` folder if it doesn't exist.

Filename: `YYYY-MM-DD_HH-MM_pr-review-<issue>.md` (use the issue/ticket from the branch
name, e.g. `devsym-319`; if none, use the branch name).

Contents:
- **Review opened** — timestamp (date + time), repo, branch, base branch, issue.
- **Dev's stated focus** — what they said they implemented.
- **Findings** — the full list from Step 3, grouped by severity.
- **Verdict** — ready / needs changes.

This file stays OPEN for the duration of the flow — the closing entry (Step 5) gets
appended to this SAME file. Tell the user the path of the file you created.

## Step 5 — Close the flow (only when the user says the PR is approved)

The flow does NOT end after the review. It ends when the user explicitly says the PR
was approved/merged.

When they do, APPEND a closing entry to the same memory file from Step 4:
- **Review closed** — timestamp (date + time).
- **Resolution** — PR approved/merged.
- **Notes** — anything relevant the user mentioned at closing (which findings were
  addressed, which were waived, etc.).

Then stop.

---

## Hard rules

- Do NOT edit, create, or delete any file EXCEPT the PR-review memory file in `.atl/memory/PR-reviews`.
- Do NOT commit or push.
- Do NOT apply fixes — only describe them.
- Do NOT modify any other file in `.atl/` (it is read-only context except for the
  memory file you write).
- Do NOT consider the task done until the user confirms the PR is approved (Step 5).