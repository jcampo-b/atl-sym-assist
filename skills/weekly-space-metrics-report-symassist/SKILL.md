---
name: weekly-space-metrics-report-symassist
description: Generate the weekly SPACE metrics report for SymAssist from the last 7 days of work (memory + git log), in Slack-ready plain-text format.
---

You are generating a weekly SPACE metrics report for Jhonny's Claude/Agents work on
**SymAssist**, in Slack-ready plain-text format.

## OBJECTIVE
A single Slack-friendly text block summarizing all work done this week with Claude on
SymAssist tickets, with totals at the end.

## STEP 1 — Review the work
Gather the week's work for the **SymAssist** project (BE / FE / AI) from these sources,
in priority order:
1. `.atl/memory/` files from the last 7 days — primary source. solve-ticket writes a
   per-ticket entry here (step 8c) with the time breakdown, model, and adoption.
2. `git log` on each repo for the date window — to catch tickets/commits and cross-check.
3. Session transcripts if available (`list_sessions`) — supplementary; do not rely on them,
   they may be absent.

For each ticket, extract:
- Linear ticket key and title (`DEVSYM-XXX`).
- Estimation (points and/or hours from the ticket).
- What was done (changes / deliverables).
- The time breakdown from the step-8c block in `.atl/memory/`.

If a ticket has no recorded times, mark it as "times not recorded" instead of inventing
them. Collect ALL such tickets and surface them in the missing-times checklist (STEP 4).

## STEP 2 — Per-task activity checklist
- Explain context to Claude: X min
- Prepare/create the PR: X min
- Review Claude's modifications: X min
- **Subtotal: X h Y min**
(No "explain context to Danny/DevOps" — that was Newbook-specific. If an analogous item
shows up in SymAssist, add it; otherwise omit it.)

## STEP 3 — Per-task SPACE metrics
- **Performance (P):** actual time vs estimated.
- **Adoption (A):** % of tasks 100% AI vs with intervention; how much was reworked.
- **Time Saved:** estimated - actual.
- **Model used:** Opus / Sonnet (optional — Opus for architecture/perf/caching/auth, Sonnet for scoped fixes/FE/discovery).

## STEP 4 — Slack format (paste EXACTLY)

---

📊 *SPACE Metrics Report — SymAssist*
Week of: [DATES]
Generated: [TODAY]

⚠️ *Before posting — tickets missing time data:* [list every DEVSYM-XXX with "times not
recorded", or write "none — all tickets have times" if complete. This line is for Jhonny
to complete the times in `.atl/memory/` and re-run, BEFORE posting to the channel. If the
list says "none", the report is ready to post as-is.]

*✅ COMPLETED TASKS*

*Ticket:* DEVSYM-XXX - Name
*Estimation:* X points (~Y h)
*Changes:* description

⏱️ *Time breakdown:*
  • Context to Claude: X min
  • Prepare PR: X min
  • Review modifications: X min
  *Total: X h Y min*

💾 *Time saved:* X h Y min
🤖 *Model:* [Opus/Sonnet]
🔧 *Adoption:* [100% AI | with intervention]

---

[REPEAT for each ticket]

---

*📈 WEEKLY TOTALS*
  • Total time spent: X h
  • Total estimated: Y h
  • *Total saved: Z h* ✨
  • PRs: X
  • Average per task: X h
  • Model(s): [list]
  • Baseline: 2x

---

## SLACK FORMAT INSTRUCTIONS
- *asterisks* for bold (not markdown with #).
- • for bullets.
- Emoji sparingly: 📊 ✅ ⏱️ 💾 🤖 📈 ✨ 🔧
- Short lines (Slack wraps at ~80 chars).
- Section breaks with ---
- A single copy-paste-able block. NO code blocks or backticks.
- All generated content (changes, adoption notes, everything) must be in English.

## HOW IT RUNS
This runs as a **local scheduled task** in Claude Code (Schedule → New task), NOT as a
cloud Routine. Reason: the data sources (`.atl/memory/`, git log) and this personal skill
live locally, so the task must run on the machine where they are — a cloud Routine clones
a repo and would not see them.

- Frequency: weekly, Friday (Anto runs 16:00; adjust to your preference).
- Work in a project or folder: `/Users/awesomejohnny/Development/Braintly/SymAssist`
  (the orchestration root, so it sees `.atl/` and all three repos).
- Model: Sonnet (read + format).
- Caveat: a local task only fires while the app is open and the machine is awake. If it's
  closed at fire time, it runs on next open and notifies you.
- Run it manually once before trusting the schedule, to verify the output.