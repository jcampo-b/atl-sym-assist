---
name: refine-ticket-symassist
description: Refine one or more SymAssist Linear tickets into implementation-ready descriptions + Claudio prompts, cross-referenced against RM docs, RESTClient contracts, and real dependencies. Writes a refining-memory artifact per ticket. Never writes to repos or Linear — Johnny reviews and pastes manually.
argument-hint: <DEVSYM-XXX> [DEVSYM-YYY ...] [be=<path>] [fe=<path>] [ai=<path>]
---

You are refining Linear tickets for SymAssist so that any developer — or Claudio via
`solve-ticket-symassist` — can pick one up without ambiguity. The refinement is grounded
in three real sources, in order of trust:

1. **RESTClient `.http` files** — the observed shape of each RM-backed endpoint. Most reliable.
2. **The trap list in §5** — distilled knowledge from past mistakes.
3. **`.atl/docs/extracted/*.md`** — RM's theoretical spec. Useful to discover endpoints you
   haven't touched yet, but verify against 1 and 2 before trusting it (RM's real behaviour
   diverges from its docs — see the `orderingOptions` 404 and the dual Virgin IDs).

Your output is **advisory and mostly read-only**. The ONE thing you write is the refining-memory
artifact (§7), under `.atl/memory/`. You do NOT write to any repo source file, do NOT update
Linear, do NOT commit. Johnny reviews every output and pastes it manually.

## 0. Load context (ALWAYS first)

Source `/Users/awesomejohnny/Development/Braintly/SymAssist/.env` to load `SYMASSIST_ROOT`.
Every `.atl/`-rooted path in this skill resolves relative to `$SYMASSIST_ROOT`, NEVER to the
repo you happen to be sitting in. If `.env` is missing or `SYMASSIST_ROOT` is unset, STOP and
ask Johnny — do not guess the root.

Then read, in this order:

1. `$SYMASSIST_ROOT/.atl/AGENTS.md` — 4-layer architecture, RM traps, conventions, git workflow.
2. `$SYMASSIST_ROOT/.atl/memory/RULES.md` — consolidated corrections and decisions. Always.
3. `$SYMASSIST_ROOT/.atl/linter.md` — so the refinement knows what CI will catch (read from
   `$SYMASSIST_ROOT`, not from a repo working dir).
4. `$SYMASSIST_ROOT/.atl/skills/code-review/SKILL.md` — the review lens; anticipate review blockers.

Repo base paths (unless the user overrides via `be=` / `fe=`):
- BE: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-Backend`
- FE: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-Frontend`
- AI: `/Users/awesomejohnny/Development/Braintly/SymAssist/SymAssist-AI-Service`

Do NOT pre-load the whole project. Everything below is read lazily, driven by each ticket.

## 1. Parse arguments

From `$ARGUMENTS`:
- **ticket_ids** — all `DEVSYM-NNN` tokens (one or more).
- **be_path** — value after `be=`, else the default above.
- **fe_path** — value after `fe=`, else the default above.
- **ai_path** — value after `ai=`, else the default above (`SymAssist-AI-Service`).

Process tickets one at a time, in the order given. Context must not bleed between tickets.

## 2. Read each ticket (Linear MCP)

For each ticket ID, fetch via the **Linear** connector:
- Title, description, acceptance criteria.
- Comments (especially grooming/refinement discussion).
- Parent issue and sub-issues, if any.
- Labels and current status.
- Linked issues or attachments (Figma, docs, `.http` files).

In 2–3 sentences, summarise what the ticket is currently asking for.

**If the ticket has no description and no meaningful title → flag it as
`UNREFINED — insufficient context` and skip to the next ticket. Do not guess.**

## 3. Locate the ticket in the codebase (module → contract map)

SymAssist BE modules map almost 1:1 to a RESTClient `.http` file. Use this as a hard lookup,
not a guess:

| Module (`app/Modules/`) | RESTClient file | RM-backed? |
|---|---|---|
| `Tasks` | `tasks.http`, `tasks-counts.http`, `subtasks.http` | yes |
| `TaskComment` | `task-comments.http` | no (SA-local) |
| `TaskActivityLog` | `activity-log.http` | no (SA-local) |
| `Units` | `units.http` | yes |
| `Properties` | (via `integrations.http`) | yes |
| `ContactInfo` | `contact-info.http` | yes |
| `Onboarding` | `onboarding.http` | yes |
| `UserProfile` | `user-profile.http` | no (SA-local) |
| `Users` | `users-creation.http` | no (SA-local) |
| `Auth` | `auth.http` | dual (Sanctum + RM token) |
| `DailyPlanner` | `daily-planner.http` | yes |
| `Dashboard` | `dashboard.http` | yes |
| `Integrations` | `integrations.http` | yes (RM lives here) |

Steps:
1. Identify the **target repo(s)**: BE / FE / AI (may be more than one).
2. For BE, identify the **module** from the table, then the **layer(s)**: `Integrations/RentManager/`
   (RM PascalCase boundary), `Modules/<Name>/` (SA snake_case domain), `Http/`. For FE, the
   feature folder, component, hook, or API-client file.
3. Read the **RESTClient `.http`** for that module first — it shows the real request/response
   shape RM expects. This is your primary contract source.
4. Then read the **likely code files** to touch: Services, Mappers, FormRequests, Controllers,
   StateMachine, React components, hooks, Zod schemas.

```bash
cat "$be_path/RESTClient/<module>.http"
cat "$be_path/app/Modules/<Module>/Services/<Service>.php"
cat "$fe_path/src/features/<feature>/<Component>.tsx"
```

Read only what is relevant. If unsure which file, list the directory first, then narrow.
Do NOT dump entire trees.

**AI-Service layout (layered, NOT module-vertical like BE).** The AI repo has no RESTClient
equivalent and no per-module folders — it is organised by horizontal layer under `src/app/`:

| Layer | Path | What lives here |
|---|---|---|
| Routers | `src/app/api/v1/<domain>.py` | FastAPI routers, one per domain. Currently `health.py` (no auth), `chat.py` (`Depends(verify_api_key)` at router level). Prefix is `/api/v1/`, wired in `main.py`. |
| Services | `src/app/services/` | Business logic. `services/backend/` calls back into the SymAssist BE; `services/tools/` is LLM tool-calling. |
| DB models | `src/app/models/db/` | SQLAlchemy async models (asyncpg + pgvector). |
| Core | `src/app/core/` | Config, dependencies, security (`verify_api_key` lives here). |
| Migrations | `src/alembic/versions/` | Alembic revisions. Deploy runs `alembic upgrade head` as a one-off task before rolling deploy. |

For an AI ticket: identify the router (or new domain), then the service, then whether it needs
a model/migration. The contract source is the Pydantic schema + the router signature — there is
NO `.http` file to read. Confirm request/response shape from the router and its response_model.

## 4. Cross-reference RM docs (only for RM-backed tickets)

If the ticket touches an RM-backed module (see table), and the `.http` doesn't fully answer
the contract, consult the extracted docs. They are plain-text markdown with `##` section
headers and per-resource tables — grep by resource name, don't read the whole file.

```bash
grep -n -i "<ResourceName>" "$SYMASSIST_ROOT/.atl/docs/extracted/01-Rent-Manager-API-Overview.md"
```

Available extracted docs:
- `01-Rent-Manager-API-Overview.md` — resource types, verbs, query params (`fields`, `embed`,
  `filters`, `pagenumber`, `pagesize`, `orderby`), pagination (auto >1000, `X-Total-Results`).
- `02-Rent-Manager-API-QuickStart.md` — resources/verbs primer, Location access model.
- `03-Partner-Network-Resource-Guide-2024.md` — integration activation, sandbox, onboarding.
- `04-Voice-AI-V3-Planning.md` — Voice Caller V3 spec (Twilio subaccounts per PM, invite flow,
  the multi-PM tables `voice_clients` / `vc_invites`, the `RentManagerHttpClient` refactor risk).

**Precedence when doc and reality disagree:** the `.http` and the §5 trap list win over the PDF
docs. If the extracted doc claims something the `.http` contradicts, note the divergence in the
refinement and trust the `.http`. RM docs describe the ideal; the tenant behaves differently.

## 5. Cross-reference traps and principles

With ticket intent + code + contract in hand, run through:

**RM traps (check for every RM-backed BE ticket):**
- Hard-delete (404 instead of soft-delete) — filter stale in SA layer, not RM side.
- Title must accompany any PATCH — carry it from the fetched raw issue.
- StatusID not at top level — always embed `Status` in RM requests.
- RM filters are AND-only; `schedule[]` one value per request.
- Deleted units appear in the active list — filter in RM layer.
- `orderingOptions` returns 404 for ServiceManagerIssues in this tenant.
- Newly created issues → `ServiceManagerStatusID: 1` → `Virgin` on read, write uses ID 56.
- Canonical status IDs: Virgin=56, Approval=52, Dispatch=53, Production=54, Review=55, Billing=18, Completed=22.
- `Hours` and other computed fields are read-only — never write them back.

**FE traps (check for every FE ticket):**
- `invalidateQueries` causes ~2 min delay — use `refetchQueries` for immediate dashboard refresh.
- `@/api/dashboard/keys` exists only on `stage`, not `dev` — dashboard-query PRs branch from `stage`.

**Backend discipline:**
- Never modify base migrations; new migration files only.
- `TaskStatus::cases()` is in declaration order — never `usort` it.
- No `StatusSequenceProvider`-style interfaces with a single implementation (no speculative abstractions).
**AI-Service traps (check for every AI ticket — this repo has a strict CI gate):**
- **Mypy strict** (`strict = true`) runs on `src/` in CI — every function needs full type
  annotations or the PR fails. No untyped defs, no implicit `Any` returns.
- **Ruff** lint + format-check gate the PR (`ruff check` and `ruff format --check` on `src/` and
  `tests/`, line-length 120, double quotes). Rules active: `E,F,W,I,N,UP,B,A,SIM,TCH`.
  Only `B008` (FastAPI `Depends()`/`Query()` in defaults) and `TC003` (SQLAlchemy `Mapped[]`) are ignored.
- **Auth is at router level, not per-endpoint.** `chat.py` sets `APIRouter(dependencies=[Depends(verify_api_key)])`.
  A new endpoint on an existing authed router inherits auth; a NEW router must add the `Depends`
  explicitly or it ships unauthenticated. Flag any new router for this.
- **Migrations are Alembic revisions**, not Laravel migrations. Never edit an applied revision —
  add a new one. Deploy runs `alembic upgrade head` as a one-off Fargate task that MUST exit 0
  before the rolling deploy; a broken migration blocks the whole deploy.
- **Test coverage gap:** the suite runs on `sqlite+aiosqlite`, so **pgvector and JSONB code paths
  are NOT covered by tests** (noted in `ci.yml`). Any ticket touching vector search or JSONB ops
  ships without a test net — call this out and require manual verification or a new integration test.
- Routes live under `/api/v1/`. Health (`/api/v1/health`) is intentionally unauthenticated.

## 6. Dependency & drift check (before finalising)

Before writing the refinement, confirm the ticket's assumptions match what's actually installed.
This is where stale assumptions get caught (e.g. a memory or an old ticket assuming PHP 8.4 when
the repo is 8.2, or a react-query version that changes the invalidate/refetch behaviour).

Read only the manifest lines that matter for this ticket:
```bash
# BE — PHP + package versions
grep -E '"php"|"laravel/framework"' "$be_path/composer.json"
# FE — React + state/query libs (react-query version gates the invalidate/refetch trap)
grep -E '"react"|"@tanstack/react-query"|"zod"' "$fe_path/package.json"
# AI — Python + framework. pyproject.toml is the manifest of record (CI runs `pip install -e ".[dev]"`,
# caches on pyproject.toml). There is no requirements.txt / poetry / uv in play. Python is pinned 3.12.
grep -E -i 'requires-python|fastapi|sqlalchemy|pgvector|alembic' "$ai_path/pyproject.toml"
```

If the ticket (or a memory/doc it relies on) assumes a version that disagrees with the manifest,
**flag it explicitly** in the refinement as `⚠️ Drift:` and refine against the manifest reality,
not the stale assumption. Do not silently inherit the wrong version.

## 7. Architecture flags (route to Anto)

Mark the ticket `⚠️ Architecture flag — route to Anto before implementation` if it touches:
- A new module boundary or cross-module contract.
- The state engine shape (transitions, status model).
- The auth model or token lifecycle (Sanctum + RM dual auth).
- A new mapper or a layer-contract change.
- The RM data model (RRM ↔ SA translation shape).

An architecture flag does not stop the refinement — it stops the *handoff*. Refine normally,
but tell Johnny to channel the decision to Anto before it goes to a dev or Claudio. These are
recommendations for Johnny to route, not decisions to make here.

## 8. Write the refining-memory artifact (the one write)

Persist one artifact per ticket so the refining history is traceable and so `solve-ticket`
can find it by ticket id (its `grep "devsym-<n>"` over `.atl/memory/` will match this file).

Path — build it exactly:
```
$SYMASSIST_ROOT/.atl/memory/<REPO>/<YYYY-MM>/<YYYY-MM-DD>-devsym-<n>-<slug>.md
```
- `<REPO>` is `BE`, `FE`, or `AI` (the primary repo; if multi-repo, the one with most of the work).
- `<slug>` is the hyphenated, lowercased ticket title.
- The filename MUST contain `devsym-<n>` so `solve-ticket`'s grep finds it.

`mkdir -p` the month directory first. Artifact contents (English, matches the sections below):
```
# DEVSYM-<n> — <title>
> Refined <YYYY-MM-DD>. Source of truth for implementation is still the Linear ticket + AGENTS.md.

## Current ask (as read from Linear)
<2-3 sentences>

## Target
Repo(s): <BE|FE|AI>  •  Module: <name>  •  Layer(s): <...>  •  RM-backed: <yes|no>
Contract source: RESTClient/<module>.http[, extracted doc section]

## Refined scope
<what IS included; what is explicitly NOT>

## Architecture decisions that apply
<specific rules from AGENTS.md that govern this>

## Traps in play
<only the ones that apply; "None identified" if none>

## Drift / flags
<⚠️ Drift or ⚠️ Architecture flag entries, or "None">

## Suggested estimate
<points + one-sentence justification>
```

This is a **work artifact**, not a Linear write and not a repo source change. It is the only
thing this skill writes. Never write to Linear or to any repo file.

## 9. Produce the two paste-ready sections (per ticket)

Output the two blocks Johnny pastes by hand.

---

### DEVSYM-NNN — <title>

#### Descripción refinada (ES)
*(Ready to paste into Linear. Spanish, warm-professional register. Apply the Linear
formatting rules at the end of this file — single newlines, no blank lines between blocks.)*

**Contexto**
[1–3 sentences: what problem this solves and why it matters now.]

**Módulos / archivos afectados**
- `<repo>/<path/to/file>` — [what changes here and why]

**Contrato RM de referencia** *(only if RM-backed)*
- `RESTClient/<module>.http` — [the endpoint(s) in play and their real shape]
- [extracted doc section, if consulted]

**Decisiones de arquitectura que aplican**
[Specific principles from AGENTS.md. "La validación va en el FormRequest, no en el Service"
beats "respetar capas".]

**Scope exacto**
[What IS included. Be explicit about what is NOT if confusion is likely.]

**Traps conocidas para este cambio**
[Only the §5 traps that apply. "None identified" if none.]

**Criterios de aceptación**
- [ ] [Verifiable, concrete. "GET /api/tasks returns snake_case fields only", not "it works".]

**Estimación sugerida**
[1 / 2 / 3 / 5 / 8 / 13 / 21, with one sentence of justification.]

---

#### Claudio prompt (EN)
*(Ready to paste into a Claude Code session running `solve-ticket-symassist`. English.
Vocabulary aligned with solve-ticket's frontier/scoped classifier — use the words
`architecture`, `mapper`, `auth`, `state`, `caching`, `write-path` when they genuinely apply,
so the classifier gates correctly.)*

```
Read in this order before doing anything:
1. .atl/AGENTS.md
2. .atl/memory/RULES.md  (+ grep devsym-<n> in .atl/memory/ for this ticket's refining artifact)
3. .atl/linter.md
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
[2–4 sentences: what this is, current behaviour, desired behaviour. Standalone.]

## Architecture decisions
[Specific AGENTS.md rules. E.g. "Validation lives in the FormRequest. Service orchestrates
only. RM PascalCase stays inside Integrations/RentManager — SA layer is snake_case throughout."]

## RM contract (if RM-backed)
[The endpoint shape from RESTClient/<module>.http so Claudio doesn't re-derive it.]

## Ticket scope
[What to implement. Exact files to touch. What NOT to touch.]

## Anti-patterns to avoid
[The §5 traps that apply, verbatim, so Claudio doesn't re-derive them.]

---

Show me the plan first. No code until I approve.
```

---

*(Repeat the block above for each ticket.)*

## 10. Closing summary

After all tickets:

| Ticket | Repo(s) | Module / Layer | RM-backed | Est. points | Flags |
|--------|---------|----------------|-----------|-------------|-------|
| DEVSYM-NNN | BE | Tasks / Service | yes | 3 | — |
| DEVSYM-MMM | FE | Feature / Hook | no | 2 | ⚠️ Architecture |

Then one paragraph on ordering dependencies between tickets (e.g. "DEVSYM-NNN must merge
before DEVSYM-MMM because the FE consumes the new endpoint"), and one line listing the
refining-memory artifacts written, by path.

## Linear ticket formatting (markdown paste)

When producing a Linear ticket description, separate block elements with SINGLE newlines,
never double (blank) lines. Linear's WYSIWYG editor treats each blank line as an extra empty
paragraph on paste, which renders with oversized line spacing.

Rules:
- One `\n` between paragraphs, headings, and the line after a heading. No blank lines.
- List items: one item per line, no blank line between items or before the list.
- Fenced code blocks are the only exception — keep them intact as-is.
- Write each paragraph as a single continuous line (let the editor wrap it); do not hard-wrap
  prose at a column width.

## Hard rules

- **One write only.** The refining-memory artifact under `.atl/memory/` (§8) is the ONLY file
  this skill writes. Do NOT write to any repo source file. Do NOT update Linear. Do NOT commit.
- **No guessing.** If a ticket is too vague to refine, flag `UNREFINED` and skip. Never invent scope.
- **No speculative abstractions.** Scope is what the ticket says. Don't expand it.
- **Contract over spec.** RESTClient `.http` and the trap list beat the extracted PDF docs.
- **Drift is surfaced, never inherited.** If a version/assumption disagrees with the manifest, flag it.
- **Architecture flags stop the handoff, not the refinement.** Mark them; tell Johnny to route to Anto.
- **One ticket at a time.** Sequential, so context doesn't bleed.

---

**Language:** ES for the refined description; EN for the Claudio prompt, the memory artifact,
and the summary table. Chat with Johnny in Spanish.