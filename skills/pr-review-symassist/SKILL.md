---
name: pr-review-symassist
description: Review-only pass on a SymAssist branch. Reads PR_REVIEW_INSTRUCTIONS.md and executes it end to end. No code, no commits, no fixes.
argument-hint: <repo: BE|FE|AI> <branch> [base-branch]
---

You are doing a REVIEW-ONLY pass for SymAssist.
You will NOT write code, commit, push, or fix anything.

## This skill is a runner, not the rules

The source of truth for what to do and how to do it is:
`.atl/PR_REVIEW_INSTRUCTIONS.md`

Read that file first — before any other step. On any contradiction between this
skill and that file, that file wins.

## What the user provides

- **Repo:** BE / FE / AI
- **Branch:** the branch the dev pushed
- **Dev's stated focus:** what they implemented / want reviewed
- (Optional) a base branch override

If any of these is missing, ask before starting.

## What this skill handles (mechanics only)

**Repo paths:**
- BE: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-Backend`
- FE: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-Frontend`
- AI: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-AI-Service`

**Base branches** (unless user overrides):
- BE → `dev`
- FE → `stage`
- AI → `dev`

**GitHub auth** — run this before any `gh` command:
```bash
gh auth switch -u jcampo-b
```

**Memory path:**
```
/Users/awesomejohnny/Development/Braintly/SymAssist/.atl/memory/PR-reviews/YYYY-MM/<REPO>/YYYY-MM-DD_pr-review-<issue>.md
```

## Execute the flow

Run every step in `PR_REVIEW_INSTRUCTIONS.md` in order:

Step 0 → read context (AGENTS.md, RULES.md, linter.md, code-review skill).
Step 1 → get the diff.
Step 1b → paste the file list in chat. STOP and paste before writing any finding.
Step 2 → review against the criteria.
Step 3 → output findings with literal code snippets (no line numbers alone).
Step 4 → write the memory file, tell Johnny the path, ask for his review time.
Step 5 → close only when Johnny confirms approval.

## Hard rules (repeated here for emphasis)

- Do NOT write a finding for a file not in the Step 1b list.
- Do NOT write a finding without the literal snippet from the file (`sed -n`).
- Do NOT consider the task done until Step 5 is complete.
- Chat with Johnny in Spanish. All review output in English.