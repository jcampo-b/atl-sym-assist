---
name: weekly-space-metrics-report-symassist
description: Generate the weekly SPACE metrics report for SymAssist from the last 7 days of work (memory + git log), in Slack-ready plain-text format.
---

You are generating a weekly SPACE metrics report for Jhonny's Claude/Agents work on
**SymAssist**, in Slack-ready plain-text format.

## OBJECTIVE
A single Slack-friendly text block summarizing all work done this week with Claude on
SymAssist tickets, with totals at the end. The output must be copy-paste-ready: pasting
it straight into Slack must require ZERO manual reformatting.

## STEP 1 — Review the work
Gather the week's work for the **SymAssist** project (BE / FE / AI) from these sources,
in priority order:
1. `.atl/memory/sessions/YYYY-MM/<REPO>/DEVSYM-<n>/space-time-log.md` — primary source.
   solve-ticket writes one of these per ticket (step 8c) with the time breakdown, model,
   and adoption. Glob `sessions/<this-month>/*/DEVSYM-*/space-time-log.md` and, if the week
   spans two months, also `sessions/<prev-month>/*/DEVSYM-*/space-time-log.md`.
   **Dating caveat:** this file has a fixed name with no date in it, unlike the session log
   sitting next to it (`YYYY-MM-DD_HH-MM_<slug>.md` in the same `DEVSYM-<n>/` folder). To
   decide if a ticket falls in the 7-day window, use the sibling session log's timestamp in
   that same folder as the anchor date, not the file's mtime (mtime is unreliable across
   syncs/pulls). If no sibling session log exists for a `space-time-log.md`, fall back to
   `git log` for that DEVSYM key to date it, and flag it in the summary if still ambiguous.
   **A month folder holds multiple weeks — never dump the whole folder.** After globbing,
   drop any ticket whose anchor date falls outside [last Friday, today]. A ticket already
   reported in a previous week's run must not reappear just because it shares the same
   `YYYY-MM` folder.
2. `.atl/memory/PR-reviews/YYYY-MM/<REPO>/YYYY-MM-DD_pr-review-<issue>.md` — PR reviews done
   this week (review-only passes), each with its own SPACE log (time spent reviewing, model,
   findings count). Structure: one folder per month, one subfolder per repo (BE/FE/AI).
   Glob `PR-reviews/<this-month>/*/*.md`, and the previous month too if the week spans two.
   **Filter by the date prefix in the filename** (`YYYY-MM-DD_pr-review-*`) — only keep
   files whose date falls in [last Friday, today]. Same rule as point 1: never include a
   review just because it's in the current month's folder if its date is from an earlier
   week.
3. `git log` on each repo for the date window — to catch tickets/commits and cross-check.
4. Session transcripts if available (`list_sessions`) — supplementary; do not rely on them,
   they may be absent.

For each solved ticket, extract:
- Linear ticket key and title (`DEVSYM-XXX`).
- **Estimate (story points) — read it from Linear, see the ESTIMATE GATE below.**
- What was done (changes / deliverables).
- The time breakdown from the step-8c block in `.atl/memory/`.

For each PR review, extract: issue, repo, time spent reviewing, model, findings count.
PR reviews are a SEPARATE category from solved tickets — never merge the two counts.

If a ticket has no recorded times, mark it as "times not recorded" instead of inventing
them. Collect ALL such tickets and surface them in the pre-post checklist (STEP 4).

### ESTIMATE GATE (mandatory — run per solved ticket, do NOT skip)
The `estimate` (story points) lives in Linear, NOT in `.atl/memory/`. For every solved
ticket, call the Linear connector `get_issue(DEVSYM-XXX)` and read the `estimate` field.
Then apply this rule exactly:
- **`estimate` present (a number):** use it verbatim, e.g. "5 points". This is the only
  source of the estimate — never read it from memory files or infer it.
- **`estimate` absent / null:** write literally `no estimate set` in the *Estimation* line
  AND add that DEVSYM-XXX to the pre-post checklist in STEP 4.

Hard prohibitions for this gate:
- Do NOT derive points from any estimation table, ticket size, priority, or effort.
- Do NOT convert points to hours or hours to points. There is no points↔hours conversion
  in this report — none is defined, so none may be invented.
- Do NOT compute a "time saved" number from a points estimate. "Time saved" is only ever
  a real hours-vs-hours comparison; with no hours estimate recorded it is `no estimate set`.

## STEP 2 — Per-task activity checklist
- Research (API / approach / docs): X min
- Context to Claude: X min
- Claude execution + PR: X min
- Review modifications (IDE): X min
- **Subtotal: X h Y min**
(No "explain context to Danny/DevOps" — that was Newbook-specific. If an analogous item
shows up in SymAssist, add it; otherwise omit it.)

## STEP 3 — Per-task SPACE metrics
- **Performance (P):** actual time vs estimated — only if an hours estimate was recorded;
  otherwise report actual time only.
- **Adoption (A):** % of tasks 100% AI vs with intervention; how much was reworked.
- **Time Saved:** estimated - actual, ONLY when a real hours estimate exists. If the only
  estimate is story points (or none), Time Saved is `no estimate set` — never derived.
- **Model used:** Opus / Sonnet (optional — Opus for architecture/perf/caching/auth,
  Sonnet for scoped fixes/FE/discovery).

## STEP 4 — Slack output template (this template IS the format — copy its spacing exactly)

The block below is the single source of truth for layout. Reproduce its line breaks and
blank lines EXACTLY: every bullet on its own line, one blank line between each ticket
block, each review block, and each major section. Emit it as plain text — no code fences,
no backticks, no markdown headings.

---

📊 *SPACE Metrics Report — SymAssist*
Week of: [DATES]
Generated: [TODAY]

⚠️ *Before posting — tickets needing attention:* [List every DEVSYM-XXX that is missing time data ("times not recorded") and/or missing its Linear estimate ("no estimate set"). Write "none — all tickets have times and estimates" if complete. This line is for Jhonny to fix the data at the source — times in `.atl/memory/`, estimates in the ticket's `estimate` field in Linear — BEFORE posting. If it says "none", the report is ready to post as-is.]

*✅ COMPLETED TASKS*

*Ticket:* DEVSYM-XXX — Name
*Estimation:* [X points | no estimate set]
*Changes:* description

⏱️ *Time breakdown:*
  • Research (API / approach / docs): X min
  • Context to Claude: X min
  • Claude execution + PR: X min
  • Review modifications (IDE): X min
  *Total: X h Y min*

💾 *Time saved:* [X h Y min | no estimate set]
🤖 *Model:* [Opus/Sonnet]
🔧 *Adoption:* [100% AI | with intervention]

---

[REPEAT the ticket block above for each ticket, with one blank line before and after each --- divider]

---

*🔍 PR REVIEWS* (review-only — separate from solved tickets)

*Review:* DEVSYM-XXX (BE/FE/AI)
  • Time spent reviewing: X min
  • Findings: N blockers, M should-fix, K nits
  • Model: [Opus/Sonnet]
  • Verdict: [approved | needs changes]

[REPEAT the review block for each review, one blank line between blocks; omit this whole section if no reviews this week]

---

*📈 WEEKLY TOTALS*
  • Tickets completed: X
  • PR reviews: X
  • Total time spent (tickets): X h Y min
  • Total time spent (reviews): X h Y min
  • Total estimated: [Y h | no estimates set]
  • *Total saved: [Z h | no estimates to compare]* ✨
  • PRs opened: X
  • Average per ticket: X h Y min
  • Model(s): [list]
  • Baseline: 2x

---

## SLACK FORMAT INSTRUCTIONS
- *asterisks* for bold (not markdown with #).
- • for bullets.
- Emoji sparingly: 📊 ✅ ⏱️ 💾 🤖 📈 ✨ 🔧
- Short lines (Slack wraps at ~80 chars).
- Section breaks with ---
- A single copy-paste-able block. NO code blocks, NO backticks, NO ``` fences anywhere.
- All generated content (changes, adoption notes, everything) must be in English.

## OUTPUT SELF-CHECK GATE (run before emitting — do not skip)
Before returning the report, verify each item. If any fails, fix it and re-check.
1. Every bullet (•) sits on its own line. No two bullets share a line joined by •.
2. There is exactly one blank line between: each completed-ticket block, each PR-review
   block, and each top-level section (COMPLETED / PR REVIEWS / WEEKLY TOTALS).
3. Every completed ticket has an *Estimation:* line that is either "N points" (read from
   Linear `estimate`) or "no estimate set" — never a made-up or converted value.
4. Every ticket with "no estimate set" or "times not recorded" appears in the pre-post
   checklist line at the top.
5. No backticks, no ``` fences, no markdown headings (#) anywhere in the output.
6. "Time saved" is a real hours figure OR "no estimate set" — never derived from points.
The output is one continuous plain-text block that pastes into Slack with zero manual
edits. When in doubt about spacing, add a newline — never collapse items onto one line.

## HOW IT RUNS
This runs as a **local scheduled task** in Claude Code (Schedule → New task), NOT as a
cloud Routine. Reason: the data sources (`.atl/memory/`, git log) and this personal skill
live locally, so the task must run on the machine where they are — a cloud Routine clones
a repo and would not see them. The Linear connector (for the ESTIMATE GATE) must be
available in the run environment.

- Frequency: weekly, Friday (Anto runs 16:00; adjust to your preference).
- Work in a project or folder: `/Users/awesomejohnny/Development/Braintly/SymAssist`
  (the orchestration root, so it sees `.atl/` and all three repos).
- Model: Sonnet (read + format).
- Caveat: a local task only fires while the app is open and the machine is awake. If it's
  closed at fire time, it runs on next open and notifies you.
- Run it manually once before trusting the schedule, to verify the output.