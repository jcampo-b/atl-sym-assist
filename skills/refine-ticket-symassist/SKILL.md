---
name: refine-ticket-symassist
description: Refine one or more SymAssist Linear tickets into implementation-ready descriptions + Claudio prompts. Read-only — no writes to repos or Linear. Produces output for Johnny to review and paste manually.
argument-hint: <DEVSYM-XXX> [DEVSYM-YYY ...] [be=<path>] [fe=<path>]
---

You are refining Linear tickets for SymAssist so that any developer — or Claudio — can
pick one up without ambiguity. Your output is **read-only**: you do NOT write to repos,
do NOT update Linear, do NOT commit anything.

## 0. Load project context (always first)

Read, in this order, before doing anything:

1. `.atl/AGENTS.md` — 4-layer architecture, RM traps, conventions, git workflow.
2. All files in `.atl/memory/` — prior corrections and decisions that must inform refinement.
3. `linter.md` of each repo that will be involved — so the refinement knows what CI will catch.
4. `.atl/skills/code-review/SKILL.md` — the review lens; use it to anticipate review blockers.

Base paths (unless the user overrides via `be=` / `fe=` arguments):
- BE: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-Backend`
- FE: `/Users/awesomejohnny/Development/Braintly/SymAssist/Sym-FE-Stage`
- AI: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-AI-Service`

## 1. Parse arguments

Extract from `$ARGUMENTS`:
- **ticket_ids** — all `DEVSYM-NNN` tokens (one or more).
- **be_path** — value after `be=`, if present; otherwise use the default above.
- **fe_path** — value after `fe=`, if present; otherwise use the default above.

Process tickets one at a time, in the order given.

## 2. Read each ticket (Linear MCP)

For each ticket ID, fetch via the **Linear** connector:
- Title, description, acceptance criteria.
- Comments (especially any grooming/refinement discussion).
- Parent issue and sub-issues, if any.
- Labels and current status.
- Linked issues or attachments (Figma, docs, `.http` files).

In 2–3 sentences, summarise what the ticket is currently asking for.

**If the ticket has no description and no meaningful title → flag it as `UNREFINED — insufficient context`
and skip to the next ticket. Do not guess.**

## 3. Infer affected modules and files

Based on the ticket title + description + your knowledge of the SymAssist architecture:

1. Identify the **target repo(s)**: BE / FE / AI (could be more than one).
2. Identify the **target layer(s)**: for BE that is one of `Integrations/RentManager`, `Modules/<Name>/`, `Http/`; for FE it is a feature folder, a component, a hook, or an API client file.
3. Identify the **likely files** to touch: Services, Mappers, FormRequests, Controllers, StateMachine, React components, hooks, Zod schemas, etc.

Then **read those files** from disk:
```bash
cat <be_path>/app/Modules/<Module>/Services/<Service>.php
cat <fe_path>/src/features/<feature>/<Component>.tsx
# etc.
```

Read only what is relevant. Do not dump entire directories. If you are unsure which file,
read the module index or the directory listing first, then narrow down.

## 4. Cross-reference with traps and principles

With the ticket intent + the code in hand, run through:

**RM traps (always check for BE tickets):**
- Hard-delete (404 instead of soft-delete) — filter stale in SA layer, not RM side.
- Title must accompany any PATCH — carry it from the fetched raw issue.
- StatusID not at top level — always embed `Status` in RM requests.
- RM filters are AND-only; `schedule[]` one value per request.
- Deleted units appear in the active list — filter in RM layer.
- `orderingOptions` returns 404 for ServiceManagerIssues in this tenant.
- Newly created issues → `ServiceManagerStatusID: 1` → `Virgin` on read, write uses ID 56.
- Canonical status IDs: Virgin=56, Approval=52, Dispatch=53, Production=54, Review=55, Billing=18, Completed=22.

**FE traps (always check for FE tickets):**
- `invalidateQueries` causes ~2 min delay — use `refetchQueries` for immediate dashboard refresh.
- `@/api/dashboard/keys` exists only on `stage`, not `dev` — dashboard-query PRs branch from `stage`.

**Backend discipline:**
- Never modify base migrations; new migration files only.
- `TaskStatus::cases()` is in declaration order — never `usort` it.
- No `StatusSequenceProvider`-style interfaces with one implementation.
- Cross-user cleanup must scan all `UserProfile` rows, not scope to `auth()->id()`.

**Architecture flags (route to Anto if present):**
- New module boundary or cross-module contract.
- Changes to the state engine shape.
- Auth model or token lifecycle changes.
- A new mapper or layer-contract change.
If any of these apply, mark the ticket with `⚠️ Architecture flag — route to Anto before implementation`.

## 5. Produce output for each ticket

Output one block per ticket with exactly these two sections:

---

### DEVSYM-NNN — <title>

#### Descripción refinada (ES)
*(Ready to paste into Linear. Spanish, warm-professional register.)*

**Contexto**
[1–3 sentences: what problem this solves and why it matters now.]

**Módulos / archivos afectados**
- `<repo>/<path/to/file>` — [what changes here and why]
- ...

**Decisiones de arquitectura que aplican**
[Which principles from AGENTS.md / Engineering Principles govern this change. Be specific: "La validación va en FormRequest, no en el Service" is better than "respetar capas".]

**Scope exacto**
[Bullet list of what IS included in this ticket. Be explicit about what is NOT included if there's a likely confusion.]

**Traps conocidas para este cambio**
[Only the traps from §4 that actually apply. Omit the ones that don't. If none apply, write "None identified."]

**Criterios de aceptación**
- [ ] [Verifiable, concrete, testable. Not "it works" — "GET /api/tasks returns snake_case fields only".]
- [ ] ...

**Estimación sugerida**
[Points from the scale: 1 / 2 / 3 / 5 / 8 / 13 / 21. One sentence justifying the pick.]

---

#### Claudio prompt (EN)
*(Ready to paste into a Claude Code session. English. Structure matches the Operating Manual.)*

```
Read in this order before doing anything:
1. .atl/AGENTS.md
2. All files in .atl/memory/
3. linter.md
4. .atl/skills/code-review/SKILL.md

---

## Ticket
DEVSYM-NNN — <title>

Branch commands:
  git fetch origin
  git checkout -b <type>/devsym-<n>-<hyphenated-title> origin/<base-branch>
  # BE base = dev | FE base = stage | AI base = dev

---

## Context
[2–4 sentences: what this ticket is about, what the current behaviour is, what the
desired behaviour is. Written so Claudio can start without reading the ticket itself.]

## Architecture decisions
[The specific rules from AGENTS.md that govern this change. E.g.:
"Validation lives in the FormRequest. Service orchestrates only. RM PascalCase stays
inside Integrations/RentManager — SA layer uses snake_case throughout."]

## Ticket scope
[What to implement. Exact files to touch. What NOT to touch.]

## Anti-patterns to avoid
[Specific traps from §4 that apply to this ticket. Pulled verbatim from the trap list
so Claudio doesn't have to re-derive them.]

---

Show me the plan first. No code until I approve.
```

---

*(Repeat the block above for each ticket.)*

## 6. Closing summary

After all tickets, output a short table:

| Ticket | Repo(s) | Layer(s) | Est. points | Flags |
|--------|---------|----------|-------------|-------|
| DEVSYM-NNN | BE | Service / Mapper | 3 | — |
| DEVSYM-MMM | FE | Feature / Hook | 2 | ⚠️ Architecture |

Then a one-paragraph note on any ordering dependencies between the tickets
(e.g. "DEVSYM-NNN must merge before DEVSYM-MMM because the FE consumes the new endpoint").

## Hard rules

- **Read-only.** Do NOT write to any repo file. Do NOT update Linear. Do NOT commit.
- **No guessing.** If a ticket is too vague to refine, flag it and skip it. Never invent scope.
- **No speculative abstractions.** Scope is what the ticket says. Don't expand it.
- **Architecture flags stop the ticket.** If it touches architecture territory, mark it and
  tell Johnny to route it to Anto before handing it to a dev or Claudio.
- **One ticket at a time.** Process them in sequence so context doesn't bleed between tickets.

---

**Language:** all output (descriptions, prompts, table) in the language specified per section
(ES for descriptions, EN for Claudio prompts and the summary table).
Chat with Johnny in Spanish.