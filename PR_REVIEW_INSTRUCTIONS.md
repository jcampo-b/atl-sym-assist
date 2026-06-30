# PR Review Instructions

> **Purpose.** This file defines the complete flow for a **review-only** pass on a
> branch. Open a chat with Claudio and say:
>
> > "Do a PR review. Repo: FE. Branch: `DEVSYM-33-build-workflow-view-ex-kanban-frontend`. The dev implemented: new Workflow page with status lanes, filters, and task detail popover."
>
> Claudio runs the full flow below: read context → diff → scope gate → review →
> record in memory → wait for approval → close.
> No code is written, committed, or pushed.

---

## Role

Review-only pass. You will NOT write code, commit, push, or fix anything.
Output: a list of actionable findings + a memory record of this review.

---

## Workspace layout

Orchestration root: `/Users/awesomejohnny/Development/Braintly/SymAssist`

Repos:
- BE: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-Backend`
- FE: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-Frontend`
- AI: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-AI-Service`

Memory dir: `/Users/awesomejohnny/Development/Braintly/SymAssist/.atl/memory`

Base branch per repo (unless user overrides):
- BE → `dev`
- FE → `stage`
- AI → `dev`

---

## Step 0 — Read context

```bash
cd <repo-path>
```

Read in this order:
1. `AGENTS.md`
2. `.atl/memory/RULES.md`
3. `linter.md`
4. `.atl/skills/code-review/SKILL.md`

If changes touch RM integration (BE only): also read `docs/rent-manager-integration.md`
and `docs/rent-manager-filters.md`.

**What this context is for:** architectural conventions, known traps, linter rules,
layer discipline. It is background that helps you evaluate the diff faster.
It is NOT a source of findings. Findings come only from the diff read in Step 1.

Do NOT grep session memory. The diff is the context for this review.

GitHub auth (before any `gh` command):
```bash
gh auth switch -u jcampo-b
```

---

## Step 1 — Get the diff

```bash
git fetch origin
git checkout <branch>
git diff origin/<base-branch>...<branch>
```

Read every changed file in the diff. This is the ground truth. Nothing outside
this diff is reviewable in this PR.

---

## Step 1b — Scope gate (MANDATORY — paste output in chat before continuing)

```bash
git diff --name-only origin/<base-branch>...<branch>
```

Paste the full output in chat as a labeled block:

```
Files in scope:
src/features/workflow/hooks/use-workflow-filters.ts
src/features/workflow/components/StatusCategoryRow.tsx
...
```

**Hard rule:** any finding whose file is not in this list is invalid and must be
discarded. Stop here and paste the list before writing a single finding.

---

## Step 2 — Review criteria

Evaluate only the files in the Step 1b list, against:

1. Dev's stated focus.
2. `.atl/skills/code-review/SKILL.md`.
3. Layer discipline: RRM → RM → SA → FE. No RM PascalCase outside
   `app/Modules/Integrations/RentManager/`.
4. Within-layer responsibilities (FormRequest / Service / StateMachine / Mapper /
   Controller).
5. Known RM traps (from `AGENTS.md` / `RULES.md`): Title on PATCH, Status embed,
   StatusID 1 → Virgin, hard-delete, no OR in filters, read-only computed fields.
6. Linter rules from `linter.md`.
7. Code quality: no dead code, no speculative abstractions, no one-off extractions
   without a second caller, no over-engineering.

Criteria 3–6 are BE-specific. For FE/AI: apply dev's focus, code-review skill,
and that repo's linter. Skip RM-layer rules.

---

## Step 3 — Output the review

For each finding, output ALL of these fields in this exact order:

---

**File:** `<repo-relative path>`
**Code snippet** (paste the exact lines from the file that you are flagging — use
`sed -n '<start>,<end>p' <path>` to get them; do not paraphrase or reconstruct):
```
<literal lines from the file>
```
**Severity:** `blocker` / `should-fix` / `nit`
**What's wrong:** one line.
**Why:** cite the rule, skill, trap, or doc it violates.
**Suggested fix:** describe the change — do NOT apply it.
**Comment for the dev:** short, polite, English — ready to paste into the PR.

---

**Why snippets instead of line numbers:**
Line numbers drift between versions and are easy to misremember. A literal snippet
is self-verifying — the dev can find it instantly with Ctrl+F, and a fabricated
snippet is immediately obvious. Never substitute a line number for a snippet. If you
cannot produce the real snippet with `sed -n`, you do not have the real location —
find it before writing the finding.

Group findings: blockers first, then should-fix, then nits.
Skip clean files — do not mention them.
End with a one-line verdict: **ready to merge** / **needs changes before merge**.

---

## Step 4 — Record in memory

Create this file (make folders if needed):
```
/Users/awesomejohnny/Development/Braintly/SymAssist/.atl/memory/PR-reviews/YYYY-MM/<REPO>/YYYY-MM-DD_pr-review-<issue>.md
```

Where:
- `YYYY-MM` = current year-month (e.g. `2026-06`)
- `<REPO>` = `BE`, `FE`, or `AI`
- `<issue>` = ticket from the branch name (e.g. `devsym-33`); if none, use branch name

Contents:
```markdown
## Review opened
- Timestamp:
- Repo:
- Branch:
- Base branch:
- Issue:

## Dev's stated focus
<what they said they implemented>

## Files in scope
<paste the Step 1b list verbatim>

## Findings
<full list from Step 3, grouped by severity, including each snippet>

## Verdict
<ready to merge | needs changes before merge>

## SPACE log
- Time spent reviewing: TBD
- Model:
- Findings: N blockers, M should-fix, K nits
- Verdict:
```

Tell Johnny the path of the file you created, then ask in Spanish:
"¿Cuánto tiempo te llevó **revisar** este PR? (Lo necesito para el SPACE log.)"

This is about Johnny's review time (reading + thinking), not implementation time.
Once he answers, update the SPACE log. If he says "skip", write "not recorded".

---

## Step 5 — Close (only when Johnny confirms the PR is approved or merged)

Append to the same memory file:
```markdown
## Review closed
- Timestamp:
- Resolution: approved / merged
- Notes: <which findings were addressed, which were waived>
```

Ask first:
"Antes de cerrar, ¿cuántos minutos te llevó leer y validar el review?
(Tu tiempo de lectura, no el mío de ejecución.)"
If he says "skip" or doesn't answer → write "not recorded". Never guess.

Then stop.

---

## Hard rules

- Do NOT edit any file except the PR-review memory file in `.atl/memory/PR-reviews/`.
- Do NOT commit, push, or apply fixes.
- Do NOT write a finding for a file not in the Step 1b list.
- Do NOT write a finding without the literal code snippet from the file.
- Do NOT consider the task done until Johnny confirms approval (Step 5).

---

## SPACE log (for weekly report)
- Time spent reviewing: X min        [Johnny-provided, ask before closing]
- Model: [Opus | Sonnet]
- Findings: N blockers, M should-fix, K nits