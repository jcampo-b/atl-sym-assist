---
name: solve-ticket-symassist
description: Solve a Linear ticket end-to-end in SymAssist (read → plan → branch → implement → self-review → PR → summaries). Bound to .atl/AGENTS.md as the brain.
argument-hint: <DEVSYM-XXX> [repo: BE|FE|AI]
---


You are solving Linear ticket **$ARGUMENTS** for SymAssist.

> This command does NOT replace `.atl/AGENTS.md` — it executes it. AGENTS.md is the source
> of truth for conventions, layers, traps, and the git workflow. This file only orchestrates
> the order of the steps. On any contradiction, AGENTS.md wins.

## 0. Load the brain (ALWAYS, before anything)

Read, in this order, before doing anything else:
1. `.atl/AGENTS.md` — conventions, 4-layer architecture, RM traps, git workflow, code quality rules.
2. All files in `.atl/memory/` — corrections and decisions from prior sessions that must NOT be repeated.
3. `linter.md` of the target repo — to avoid breaking the linter when implementing.
4. `.atl/skills/code-review/SKILL.md` — the checklist you will self-review against in step 5.

GitHub: before any `gh` command, run `gh auth switch -u jcampo-b`.

## 1. Read the ticket (Linear MCP)

Fetch ticket **$ARGUMENTS** via the **Linear** connector (not Jira). Read:
- Description and acceptance criteria.
- Comments (including anything discussed in refinement/grooming).
- Parent issue and sub-issues, if any.
- Attachments and links (Figma, docs, referenced RESTClient `.http` files).

In 2-3 sentences, state what needs to be done.

**Information gate:** if the ticket has no description, no clear criteria, or there is an
unanswered comment that changes scope → STOP and send all questions together.
This gate always applies.

## 2. Plan + conditional plan gate

Build an execution plan with the context from steps 0 and 1.

**Classify the ticket** by these signals (in title, description, labels, or expected diff):

- **Frontier / non-trivial** if it touches: architecture, a modeling decision (RRM vs SA),
  caching, state, performance, the dual auth systems (Sanctum + RM token), a new mapper or
  a layer-contract change, or if the research knowledge source is "it depends". Keywords:
  `caching`, `state`, `auth`, `architecture`, `mapper`, `subtask`, `write-path`, `threshold`,
  `exception`.
  → **Show the plan and wait for human approval before writing a single line.**

- **Scoped** if it is: CRUD, a scoped FE fix, a copy/label change, read-only discovery, a
  FormRequest validation tweak, simple inline mapping.
  → Run straight to implementation, posting brief status between phases.

If unsure of the classification, treat it as frontier and show the plan. Task research is
the biggest technical challenge in SymAssist and is not automated — when research is
involved, always gate.

## 3. Prepare the branch

Follow the "Git workflow / Branch" section of AGENTS.md EXACTLY:
- Confirm working dir is under `/Users/awesomejohnny/Development/Braintly/SymAssist`.
- `git status` — if there are uncommitted changes that look like WIP, STOP and ask. Don't stash.
- Base branch per repo: **BE=`dev`, FE=`stage`, AI=`dev`**.
- `git fetch origin && git branch -a | grep devsym-<n>` to check if a branch already exists.
- Decide reuse / new-typed-branch / create-from-base per the 3 rules in AGENTS.md.
- Naming: `<type>/devsym-<n>-<hyphenated-title>` with `feat` or `fix`. Never branch from another feat/fix.

## 4. Implement

Read the code first, then change it. Respect:
- The 4 layers (RRM → RM → SA → FE). Zero RM PascalCase outside `app/Modules/Integrations/RentManager/`.
- Within-layer: FormRequest = validation, Service = orchestration, StateMachine = transitions, Mapper = translation, Controller = thin.
- The AGENTS.md traps (Title on PATCH, embed Status, StatusID 1 → Virgin, hard-delete, no OR in filters, read-only computed fields like `Hours`).
- Migration discipline: never touch base migrations, always a new file.

Conventional commits with the key in scope, one logical unit per commit (core / wiring / tests separate):
- `feat(DEVSYM-<n>): ...`, `fix(DEVSYM-<n>): ...`, `refactor(...)`, `test(...)`, `chore(...)`.
- Imperative subject, lowercase, < ~70 chars.
- If a commit touches > ~8 files, stop and ask whether to split it.

**Commit authorship: NEVER add `Co-Authored-By` or any attribution to Claude.**
(AGENTS.md forbids it explicitly — this replaces the Newbook version's footer.)

If there is a test suite, run the relevant tests before pushing. Don't block on tests
already failing on `dev`/`stage`, but flag them in the summary.

## 5. Self-review + human review gate

Run the `.atl/skills/code-review/SKILL.md` checklist against every file you touched.
Save relevant findings to `.atl/memory/` if something comes up worth not repeating.

Then STOP. Don't commit, don't open a PR. Notify:
"Implementation is ready. Please review the code in your IDE before I commit. Let me know when to proceed."

Wait for explicit confirmation. (This pre-commit gate is from AGENTS.md and ALWAYS applies,
even on scoped tickets.)

## 6. Sync, push, and PR

After confirmation:
- Sync with the base: `git fetch origin && git merge origin/<base>`. If there are conflicts,
  resolve them, run `docker compose exec app php artisan test`, and confirm the pre-existing
  failures were already on the base.
- `git push -u origin <branch>`.
- `gh pr create --base <base>` (BE/AI=dev, FE=stage).
- PR title: `<type>(DEVSYM-<n>): <subject>`.
- **PR body: paste MANUALLY** (no inline `--body`, it corrupts backticks). Use
  `.atl/pull_request_template.md` and the guidance in step 7.
- If the PR changes API surface, the Testing Endpoints section is MANDATORY.

## 7. PR description — readable for HUMANS

The PR is read by Jhonny or Jona to review, not by an AI. Rule of thumb: concise and actionable.

Follow `.atl/pull_request_template.md`. Specifically:
- **Description:** 1-2 sentences — what it does and why (the non-obvious "why" from the diff).
- **Changes Made:** 4-5 bullets per behavioral change, NOT per file. No implementation details.
- **Testing Endpoints:** one block per affected endpoint. For each: the **happy path**
  (request + expected response) and at least one **error case** (request + response + code).
  The request should reflect what's in the code.
- NO kilometer-long tables. NO Postman screenshots. NO long prose.
- `Additional Notes`: only decisions, known gaps, or follow-ups. Delete the section if empty.

## 8. Two summaries + time logging (SPACE)

**8a. Non-technical summary — in the Linear ticket** (comment), ALWAYS in English:
```
PR: <url>

<2-4 plain-English sentences for PM/QA/client. What was done and why. No jargon, no file paths.>
```

**8b. Internal summary — in chat for the dev:** if there was anything odd to watch for on review.

**8c. Time logging for the SPACE report.** Record at close, in the internal comment or in
`.atl/memory/`, this task's breakdown (this feeds Friday's report):
```
DEVSYM-<n> — <title>
Estimation: <ticket points / hours>
Actual time:
  • Explain context to Claude: X min
  • Prepare/create the PR: X min
  • Review Claude's modifications: X min
  • Subtotal: X h Y min
Adoption: [100% AI | with intervention] — what was reworked, if anything.
```

## 9. Memory

When done, write an entry to `.atl/memory/` (`YYYY-MM-DD_HH-MM_<slug>.md`):
What was done / Files touched / Decisions made / Open questions.
If there was a correction or review finding, a separate file `correction-<slug>.md`:
What was wrong / Correct approach / Rule going forward.

---

**Language:** all markdown, PR, commits, and summaries in **English**. The conversation
with the dev stays in Spanish.