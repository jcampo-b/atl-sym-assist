---
name: pr-review-symassist
description: Review-only pass on a SymAssist branch (read context → diff → review → record in memory → wait for approval → close). Bound to .atl/PR_REVIEW_INSTRUCTIONS.md. No code, no commits, no fixes.
argument-hint: <repo: BE|FE|AI> <branch> [base-branch]
---

You are doing a REVIEW-ONLY pass for SymAssist. You will NOT write code, commit, push, or
fix anything. Your output is a list of actionable review comments plus a memory record.

> This command does NOT replace `.atl/PR_REVIEW_INSTRUCTIONS.md` — it executes it end to end.
> That file is the source of truth for the flow and the hard rules. On any contradiction, it wins.

## What the user provides
- **Repo:** BE / FE / AI.
- **Branch:** the branch the dev pushed.
- **Dev's stated focus:** what they implemented / want reviewed.
- (Optional) a base branch override.

If any of these is missing, ask before starting.

## Step 0 — Read context first
`cd` into the correct repo, then read, before anything:
1. `AGENTS.md`
2. All files in `.atl/memory/`
3. `linter.md`
4. `.atl/skills/code-review/SKILL.md`
If the changes touch RM integration (BE), also read `docs/rent-manager-integration.md`
and `docs/rent-manager-filters.md`.

GitHub: before any `gh` command, run `gh auth switch -u jcampo-b`.

## Step 1 — Get the diff
Base branch per repo: **BE=`dev`, FE=`stage`, AI=`dev`** (unless the user overrode it).
```
cd <repo-path>
git fetch origin
git checkout <branch>
git diff origin/<base-branch>...<branch>
```
Review every changed file.

## Step 2 — Review criteria (in order)
1. The dev's stated focus.
2. `.atl/skills/code-review/SKILL.md`.
3. Layer discipline: RRM → RM → SA → FE. No RM PascalCase outside
   `app/Modules/Integrations/RentManager/`.
4. Within-layer responsibilities (FormRequest / Service / StateMachine / Mapper / Controller).
5. Known RM traps (Title on PATCH, Status embed, StatusID 1 → Virgin, hard-delete,
   no OR in filters, read-only computed fields).
6. Linter rules in `linter.md`.
7. Code quality (no over-engineering, no dead code, no speculative abstractions,
   no one-off extractions).
(Criteria 3–6 are BE-specific. For FE/AI, apply the dev's focus, the code-review skill,
that repo's conventions and linter; skip RM-layer rules that don't apply.)

**Balance:** be balanced between blockers and nitpicky comments. A blocker is something that
must be fixed before merge (broken contract, layer violation, RM trap, linter failure). A
should-fix is a clear improvement that warrants fixing before merge but is not a regression.
A nit is a good practice worth noting but non-blocking — flag it and move on, do not delay
the merge over it. The goal is to help the dev improve and learn, not to gatekeep.

## Step 3 — Output the review
Only review comments. Per finding, output ALL of these fields:

- **File + line** (e.g. `app/Modules/Tasks/Services/TaskService.php:42`)
- **Severity:** `blocker` / `should-fix` / `nit`
- **What's wrong** — one line.
- **Why** — cite the rule, skill, trap, or doc it violates.
- **Suggested fix** — what to change, described, NOT applied.
- **Comment for the dev** — a short, polite, English comment ready to paste into the PR.
  Written as if you are talking directly to the dev. E.g.:
  "This should live in the RM layer — SA modules shouldn't know RM field names.
  Consider moving this mapping to `RentManagerIssueMapper`."

Group findings by severity (blockers first, then should-fix, then nits).
Skip clean files — don't mention them.
End with a one-line verdict: **ready to merge** / **needs changes before merge**.

## Step 4 — Record the review in memory
Create the memory file at this path (create folders if they don't exist):
`/Users/awesomejohnny/Development/Braintly/SymAssist/.atl/memory/PR-reviews/YYYY-MM/<REPO>/YYYY-MM-DD_HH-MM_pr-review-<issue>.md`

Where:
- `YYYY-MM` is the current year-month (e.g. `2026-06`). One folder per calendar month.
- `<REPO>` is `BE`, `FE`, or `AI` — matching the repo the user provided.
- `<issue>` comes from the branch name (e.g. `devsym-339`); if none, use the branch name.

Example: `/Users/awesomejohnny/Development/Braintly/SymAssist/.atl/memory/PR-reviews/2026-06/BE/2026-06-24_17-00_pr-review-devsym-350.md`

Contents:
- **Review opened** — timestamp, repo, branch, base branch, issue.
- **Dev's stated focus.**
- **Findings** — full list from Step 3, grouped by severity.
- **Verdict** — ready / needs changes.
- **SPACE log** — for Friday's report:

```
PR review — <issue> (<repo>)
Time spent reviewing: TBD         ← ask Jhonny after writing the review (see below)
Model: [Opus for perf/caching/state/architecture | Sonnet for scoped/FE]
Findings: N blockers, M should-fix, K nits.
Verdict: [ready to merge | needs changes]
```

Tell the user the path of the file you created.

Then ask — in Spanish, since the conversation is in Spanish:
"¿Cuánto tiempo te llevó **revisar** este PR? (Lo necesito para el SPACE log.)"

This question is about REVIEW time (Jhonny's time reading the diff, thinking, and writing
comments) — NOT implementation time. Jhonny is the reviewer here, not the dev who built
the feature. Never ask about "implementar" or "cambios" in this context.

Once Jhonny answers, update the SPACE log in the memory file with the real time.
This file stays OPEN until Step 5.

## Step 5 — Close the flow (only when the user confirms the PR is approved/merged)
APPEND to the same memory file:
- **Review closed** — timestamp.
- **Resolution** — approved / merged.
- **Notes** — which findings were addressed, which were waived.
Then stop.

## Hard rules
- Do NOT edit, create, or delete any file EXCEPT the PR-review memory file in `.atl/memory/`.
- Do NOT commit, push, or apply fixes — only describe them.
- Do NOT consider the task done until the user confirms approval (Step 5).

---
**Language:** review comments, memory, and all deliverables in **English**.
Chat with Jhonny in Spanish.