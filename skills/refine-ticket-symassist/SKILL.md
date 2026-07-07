---
name: refine-ticket-symassist
description: Refine one or more SymAssist Linear tickets into implementation-ready descriptions + Claudio prompts, cross-referenced against RM docs, RESTClient contracts, and real dependencies. Writes a refining-memory artifact per ticket. Never writes to repos or Linear — Johnny reviews and pastes manually.
argument-hint: <DEVSYM-XXX> [DEVSYM-YYY ...] [be=<path>] [fe=<path>] [ai=<path>]
---

You are refining Linear tickets for SymAssist so that any developer — or Claudio via
`solve-ticket-symassist` — can pick one up and implement it with **zero research and zero open
doubts about what to build.** That is the bar: if a dev would need to go ask someone, dig through
old tickets, or guess at intent, the refinement isn't done yet — either the answer belongs in the
ticket, or the question does (see the escalation rule below). The refinement is grounded in three
real sources, in order of trust:

1. **RESTClient `.http` files** — the observed shape of each RM-backed endpoint. Most reliable.
2. **The trap list in §6** — distilled knowledge from past mistakes.
3. **`.atl/docs/extracted/*.md`** — RM's theoretical spec. Useful to discover endpoints you
   haven't touched yet, but verify against 1 and 2 before trusting it (RM's real behaviour
   diverges from its docs — see the `orderingOptions` 404 and the dual Virgin IDs).

**Escalate, never guess.** If refining a ticket surfaces something the ticket, the code, and the
traps genuinely can't answer — an old ticket description that conflicts with current reality, an
ambiguous business rule, a scope question — that is not this skill's call to resolve quietly. It
becomes a visible, addressed question left IN the ticket (§5), so whoever reads it in Linear
knows exactly who needs to weigh in: Johnny for anything architecture/technical-shape, Lean for
scope/priority/product intent, or "product" generically if neither fits. A ticket that silently
picked one reading of an ambiguity is worse than one that visibly asks — it just moves the
research burden onto the dev instead of removing it.

Your output is **advisory and mostly read-only**. The ONE thing you write is the refining-memory
artifact (§10), under `.atl/memory/`. You do NOT write to any repo source file, do NOT update
Linear, do NOT commit. Johnny reviews every output and pastes it manually.

**Design principle for this skill:** every check below produces a written artifact (a ledger,
a list, a flag), not just an impression. If a check ran but its finding never appears literally
in the final output, treat that as the check having failed. Prose diligence is not verification —
the same lesson `pr-review-symassist` already learned about hallucinated findings applies here:
structural gates catch what "I'll remember to mention it" doesn't.

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

**Build a comment ledger now, before moving on.** List every comment on the ticket, in order,
each with a one-line paraphrase. This ledger is not optional bookkeeping — §5 requires you to
close out every row, and §9 checks that you did. A comment that is never mentioned again by §5
is a comment that got silently dropped, which is exactly the failure mode this step exists to prevent.

**If a refining-memory artifact already exists for this ticket (from an earlier run), it is
reference material — the same as anything under `sessions/` — never a verified starting point.**
A prior artifact passing this skill's gates in an earlier run is not evidence it would pass them
now: this skill has no way to confirm the quote pairs behind an old artifact's contradictions were
ever real, and reusing it skips §5 through §9 entirely. Read it if you want the historical context,
but run §2 through §11 in full regardless, from the actual ticket text, every time. Write a new
dated artifact; do not treat an old one as already-done work.

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

**Precedence when doc and reality disagree:** the `.http` and the §6 trap list win over the PDF
docs. If the extracted doc claims something the `.http` contradicts, note the divergence in the
refinement and trust the `.http`. RM docs describe the ideal; the tenant behaves differently.

## 5. Internal consistency audit (before cross-referencing traps)

The ticket itself is not a single coherent source — it's Context + Scope + Acceptance Criteria +
a comment thread, usually written at different times by different people. Treat it the way you'd
treat conflicting RM docs vs `.http` files: don't silently pick the version that's easiest to
implement. Do this explicitly, in writing, before moving to §6:

**First, distinguish real ambiguity from freedom the ticket already gave.** Language like "X or Y",
"either... or", "e.g.", "such as" is the ticket author deliberately leaving the choice open, not an
unresolved question. Escalating an already-granted choice back to Lean or Johnny adds a redirect for
something the dev could just decide — it's the same failure as inventing a contradiction: it costs
someone a trip for nothing. The test: if you can point to the specific word that grants the freedom
("or", "e.g.") and both readings would satisfy the ticket as written, it's implementation freedom —
note it once in "Scope exacto" as "either approach is acceptable," and do NOT list it as an open
question. Only escalate when the ticket does NOT leave room for either answer, or when the two
readings would produce genuinely different behavior the ticket doesn't clearly sanction (that's a
real contradiction, subject to the quote-pair rule in point 1 below).

1. **Diff Context against Scope against Acceptance Criteria.** Look specifically for cases where
   one section names more options, states, or values than another (e.g. Context describes three
   paths/states/categories, Scope or AC only enumerate two).

   **A contradiction is only real if you can quote it.** For each one you report, copy the exact
   phrase from each section side by side — not a paraphrase, not your summary of what it implies:
   ```
   Context: "<verbatim phrase>"
   Scope:   "<verbatim phrase>"
   → these disagree because: <one line>
   ```
   If you can't produce two literal quotes that actually conflict, it isn't a contradiction —
   don't report one. This is the same discipline `pr-review-symassist` uses for line-number
   findings: unverified pattern-matching ("this reads like the same 3-vs-2 shape I found earlier")
   produces false positives that cost the reader a research trip — exactly what this skill exists
   to prevent. Re-read the two phrases once more before writing them down; "reminds me of" is not
   "is."
2. **Close out every row of the comment ledger from §2.** For each comment, decide and record one
   of exactly three outcomes:
   - **Incorporated** — folded into the refined scope; say where.
   - **Explicitly excluded** — out of scope for this ticket; say why, and whether it implies a
     follow-up ticket.
   - **Open question** — introduces a new requirement, data need, or ambiguity the ticket doesn't
     resolve. Do not resolve it yourself by guessing; surface it, tagged per the rule below.
   A comment cannot be left unclassified. If a comment describes a capability that isn't reflected
   anywhere in the current data model or scope you're about to write (e.g. a field with nowhere to
   live), that is almost always an open question, not something to quietly omit.
3. **Tag every open question and contradiction with a suggested addressee.** Whoever picks up the
   ticket needs to know who to ask, not just that a question exists:
   - `→ Johnny` — the decision changes the *system's shape*: a module boundary, the data model, the
     auth model, the state engine. Reserve this for genuine architecture calls, not for anything
     that merely *touches* a field that architecture happens to depend on.
   - `→ Lean` — scope, priority, product intent, or **what the ticket's own words mean** (a list that
     might imply 2 or 3 values, a phrase that could be read two ways). Wording ambiguity in the
     ticket text is a Lean/product question even when the eventual answer will feed into a schema —
     don't tag it `→ Johnny` just because the downstream artifact is technical. Ask: is the question
     "what did the ticket mean?" (→ Lean) or "how should the system be shaped?" (→ Johnny)? Only the
     second one is architecture.
   - `→ product` — anything neither fits (e.g. a business rule only Doug or a domain owner can settle).
   If genuinely unsure which bucket, default to `→ Lean` — routing a question to the wrong person
   costs a redirect; leaving it unaddressed costs the dev a research trip, which is the thing this
   skill exists to prevent.
4. **Output of this step**: a short "Contradictions found" list — each entry carrying its two
   verbatim quotes, per point 1 — and a completed comment ledger, each entry carrying its
   addressee tag. Both get carried forward into §10 (memory artifact) and §11 (paste-ready output)
   verbatim — this step is worthless if its findings evaporate before the final write-up. The
   quotes themselves don't need to survive into §11 (that block should read cleanly for a dev),
   but they MUST survive into §10's "Contradictions / open questions" section in full, exactly as
   the template there specifies — that section is the one place Johnny can audit a finding without
   re-deriving it himself. An internal judgment that a contradiction is real, with no quote pair
   written anywhere he can see, does not satisfy this step.

If this step finds nothing, say so explicitly ("No contradictions found; all comments incorporated
or excluded — see ledger") rather than omitting the section. An empty section reads as "not checked."

## 6. Cross-reference traps and principles

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

## 7. Dependency & drift check (before finalising)

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

**Keep the raw grep output.** You need it verbatim for the gate in §9 — this step is a source of
truth, not just a sanity check you discard once satisfied.

**A confirmed drift is not optional in the output.** If this step finds a real disagreement —
not just a stale assumption in the ticket itself, but a project doc (e.g. `AGENTS.md`)
contradicting the manifest — it must show up as `⚠️ Drift` in the §12 Flags column and in the
artifact's `Drift / flags` section, every single time, regardless of whether it changes this
ticket's implementation. It doesn't need the full architecture banner from §8 (that would be
noise on tickets where the drift is incidental to the work), but it cannot be confirmed in your
reasoning and then quietly dropped before publishing — same failure mode §5 and §8 exist to
prevent. If the drift lives in a shared doc rather than the ticket, add a one-line
"Housekeeping" note to the artifact recommending the doc get corrected — separate from the
ticket's own scope, so it doesn't get lost inside an unrelated ticket's history.

## 8. Escalation flags (architecture decisions Johnny must make consciously)

Mark the ticket `⚠️ Decision needed — architecture/boundary call before implementation` if it touches:
- A new module boundary or cross-module contract.
- The state engine shape (transitions, status model).
- The auth model or token lifecycle (Sanctum + RM dual auth).
- A new mapper or a layer-contract change.
- The RM data model (RRM ↔ SA translation shape).

Johnny owns architecture decisions for SymAssist. This flag isn't about finding someone else to
decide — it's about making sure a decision like this reaches him explicitly, as a decision, instead
of getting silently baked into a Claudio prompt as an assumed fact. Refine normally, but don't let
a new boundary slip through disguised as routine implementation detail.

**This flag is not satisfied by a mention in prose.** If any of the five triggers above apply,
§11 requires a literal banner line at the top of both paste-ready blocks and a non-empty entry
in the Flags column of the §12 summary table. Writing "this introduces a new module" inside the
"Decisiones de arquitectura" section and nowhere else does not count — that reads as a settled
technical decision, not an open decision point. If you catch yourself phrasing a new boundary as
a foregone conclusion for Claudio to build, stop and convert it into a flagged decision instead.

## 9. Pre-publish verification gate (run before writing §10/§11 — do not skip)

This is the step that makes §5, §7, and §8 actually count. Before producing any output, answer
each of the following against your own draft, literally, one by one. Treat a "no" as a blocker —
go back and fix the draft, don't publish around it.

1. Does the comment ledger from §2/§5 have every comment closed out as incorporated / excluded /
   open question? (If any comment is unaccounted for: stop, classify it.)
2. Did the §5 contradiction check turn up anything, and if so, does every contradiction found
   appear as an explicit open question in §11 — not silently resolved toward whichever reading
   is simpler to implement? **And for each one, re-check the two verbatim quotes from §5 against
   the actual ticket text one more time here** — do they really conflict, or did an earlier real
   contradiction (like a genuine 3-vs-2 split elsewhere in the ticket) pattern-match onto a section
   that actually says the same thing twice? If you can't re-find both quotes verbatim in the
   ticket right now, drop the contradiction — it doesn't ship. And confirm the quote pair is
   actually written into §10's "Contradictions / open questions" section, not just held in your
   own reasoning — if you can't produce the two quotes to write down, that itself is the signal
   the contradiction isn't real.
3. Does every version-specific claim you're about to write (a PHP/Python/package version, a
   version-gated behaviour) trace back to a literal line in the §7 grep output? If you can't point
   to the line, delete the version number and use version-agnostic language instead (e.g. "match
   `UserProfile.php`'s style" rather than inventing a PHP version). And if §7 found a genuine
   drift (a doc disagreeing with the manifest), does it appear as `⚠️ Drift` in the §12 table and
   in the artifact — not just in your reasoning?
4. If any §8 trigger applies, is there a banner in both paste-ready blocks AND a populated Flags
   cell in the §12 table? A decision flag that only exists in your reasoning and not in the
   output has failed.
5. Is the estimate justified by something concrete in scope (new boundary, RM round-trips, number
   of endpoints) rather than a general impression of size?
6. **Self-consistency of your own draft.** Read Scope, Architecture decisions, RM contract, and
   Estimate as if you were seeing them for the first time, side by side. Does any claim in one
   section contradict a claim in another? The classic version: saying "no RM round-trips" in the
   estimate while the scope section requires a cross-module call into an RM-backed module that has
   no local table — that call IS a round-trip, whether or not you named it one. §5 only audits the
   *ticket's* internal consistency; it does not check the text you are about to publish. That check
   is this one. If you find a contradiction, fix the draft — don't let both claims stand.
7. **Freedom vs. ambiguity, one more pass.** For each item in "Open questions," check it against
   the §5 rule: does the ticket already say "or" / "either" / "e.g." for this exact point? If so,
   it's implementation freedom, not an open question — move it into "Scope exacto" as a one-line
   note and drop it from the escalation list. An item that quotes the ticket's own permissive
   language and still asks someone to resolve it has failed this check.

Only proceed to §10 once all seven are satisfied for this ticket.

## 10. Write the refining-memory artifact (the one write)

Persist one artifact per ticket so the refining history is traceable and so `solve-ticket`
can find it by ticket id (its `grep "devsym-<n>"` over `.atl/memory/` will match this file).

Path — build it exactly:
```
$SYMASSIST_ROOT/.atl/memory/refinement/<REPO>/<YYYY-MM>/<YYYY-MM-DD>-devsym-<n>-<slug>.md
```
- The `refinement/` segment is fixed — it exists to keep refining-memory artifacts visually
  separate from `sessions/` and `PR-reviews/` under the same `.atl/memory/` root.
- `<REPO>` is `BE`, `FE`, or `AI` (the primary repo; if multi-repo, the one with most of the work).
- `<slug>` is the hyphenated, lowercased ticket title.
- The filename MUST contain `devsym-<n>` so `solve-ticket`'s grep finds it. A recursive
  `grep "devsym-<n>" .atl/memory/` still finds it under the new subfolder — nothing else
  needs to change on the `solve-ticket-symassist` side for this to keep working.

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

## Comment ledger
<every comment, one line each, tagged Incorporated / Excluded / Open question>

## Contradictions / open questions
<from §5; "None found" if genuinely none. For each contradiction, include the literal quote pair
that justifies it — not a paraphrase:
Context: "<verbatim phrase>"
Scope: "<verbatim phrase>"
→ <addressee tag> — <one line on why these two actually disagree>
If a candidate contradiction turns out to be the same phrase quoted twice (i.e. the two quotes
match), that's proof it isn't a contradiction — drop it, don't list it. This section exists so
Johnny can audit the finding in seconds by reading the two quotes side by side, instead of taking
the model's word for it.>

## Architecture decisions that apply
<specific rules from AGENTS.md that govern this>

## Traps in play
<only the ones that apply; "None identified" if none>

## Drift / flags
<⚠️ Drift or ⚠️ Architecture flag entries, each traceable to §7/§8, or "None">

## Housekeeping (optional)
<only if a drift lives in a shared doc like AGENTS.md rather than the ticket itself — one line
recommending the doc get corrected, kept separate from this ticket's own scope>

## Suggested estimate
<points + one-sentence justification>
```

This is a **work artifact**, not a Linear write and not a repo source change. It is the only
thing this skill writes. Never write to Linear or to any repo file.

## 11. Produce the two paste-ready sections (per ticket)

Output the two blocks Johnny pastes by hand.

If §8 flagged this ticket, **open with the banner** before anything else, in both blocks:
```
⚠️ DECISION NEEDED — architecture/boundary call before implementation: <one-line description of
the decision, e.g. "new OperativeCosts module boundary + task_id data model (no local Tasks
table)">. Johnny: confirm this consciously before a dev or Claudio builds against it.
```

**Language and structure of the Linear-facing block: match the original ticket, not a fixed default.**
Detect the language the ticket's own description was written in (not the comments — comments can be
in either language regardless of the ticket body). Write the "Descripción refinada" block in that
same language, and keep the section labels and order below exactly as they are (the structure this
skill produces doesn't change) — only the language of the labels and content switches:

| Section (ES) | Section (EN) |
|---|---|
| Descripción refinada | Refined description |
| Contexto | Context |
| Módulos / archivos afectados | Modules / files affected |
| Contrato RM de referencia | RM reference contract |
| Decisiones de arquitectura que aplican | Architecture decisions that apply |
| Scope exacto | Exact scope |
| Preguntas abiertas / contradicciones detectadas | Open questions / contradictions detected |
| Traps conocidas para este cambio | Known traps for this change |
| Criterios de aceptación | Acceptance criteria |
| Estimación sugerida | Suggested estimate |

If a ticket is genuinely bilingual or ambiguous (e.g. an English body with a Spanish title), default
to Spanish per team convention and note the ambiguity in one line — don't silently pick either.

---

### DEVSYM-NNN — <title>

#### Descripción refinada / Refined description
*(Ready to paste into Linear as-is. Language matches the original ticket per the table above.
This block MUST be delivered inside a fenced ```markdown code block — copy it out of the fence
and paste directly into Linear, no manual fixing of line breaks. Inside the fence, follow the
Linear formatting rules at the end of this file exactly: single `\n` between every paragraph,
heading, and list item — NEVER a blank line, not even between sections. A blank line inside the
fence becomes an oversized empty paragraph on paste, which is the exact problem this fence exists
to avoid. Do not let the chat UI's own markdown rendering reformat this — the fenced block is
rendered as literal text precisely so what Johnny copies matches what he sees, byte for byte.)*

The structure inside the fence, all single-spaced, no blank lines between any of the lines below:
```markdown
### DEVSYM-NNN — <title>
**Contexto / Context**
[1–3 sentences: what problem this solves and why it matters now.]
**Módulos / archivos afectados / Modules / files affected**
- `<repo>/<path/to/file>` — [what changes here and why]
**Contrato RM de referencia / RM reference contract** *(only if RM-backed)*
- `RESTClient/<module>.http` — [the endpoint(s) in play and their real shape]
**Decisiones de arquitectura que aplican / Architecture decisions that apply**
[Specific principles from AGENTS.md, one continuous line, let it wrap naturally — do not
hard-wrap at a column width. If any decision is itself an §8 trigger, point to the banner
instead of stating it as settled.]
**Scope exacto / Exact scope**
[What IS included. Be explicit about what is NOT if confusion is likely.]
**Preguntas abiertas / contradicciones detectadas / Open questions / contradictions detected**
[From §5, each with its addressee tag (→ Johnny / → Lean / → product), one line per item, no
quote pairs here — those stay in the memory artifact. "Ninguna" / "None" only if genuinely none.]
**Traps conocidas para este cambio / Known traps for this change**
[Only the §6 traps that apply. "None identified" if none.]
**Criterios de aceptación / Acceptance criteria**
- [ ] [Verifiable, concrete. "GET /api/tasks returns snake_case fields only", not "it works".]
**Estimación sugerida / Suggested estimate**
[1 / 2 / 3 / 5 / 8 / 13 / 21, with one sentence of justification.]
```
If §8 flagged this ticket, the `⚠️ DECISION NEEDED` banner line goes INSIDE the fence too, as the
first line, above the `### DEVSYM-NNN` title — it needs to survive the copy-paste into Linear, not
just sit in the chat response around the fence.

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

## Open questions / unresolved contradictions
[From §5, verbatim, if any. If genuinely none, state "None — ticket sections and comments are
consistent." Do not omit this section to save space.]

## Architecture decisions
[Specific AGENTS.md rules. E.g. "Validation lives in the FormRequest. Service orchestrates
only. RM PascalCase stays inside Integrations/RentManager — SA layer is snake_case throughout."
Anything still pending sign-off per the banner above stays out of this section — don't let it
slide back in as an assumed decision.]

## RM contract (if RM-backed)
[The endpoint shape from RESTClient/<module>.http so Claudio doesn't re-derive it.]

## Ticket scope
[What to implement. Exact files to touch. What NOT to touch.]

## Anti-patterns to avoid
[The §6 traps that apply, verbatim, so Claudio doesn't re-derive them.]

---

Show me the plan first. No code until I approve.
```

---

*(Repeat the block above for each ticket.)*

## 12. Closing summary

After all tickets:

| Ticket | Repo(s) | Module / Layer | RM-backed | Est. points | Flags |
|--------|---------|----------------|-----------|-------------|-------|
| DEVSYM-NNN | BE | Tasks / Service | yes | 3 | — |
| DEVSYM-MMM | FE | Feature / Hook | no | 2 | ⚠️ Architecture |

The Flags column is not decorative — if §8 triggered for a ticket, it MUST show
`⚠️ Architecture` here, matching the banner in §11. If §5 found unresolved contradictions,
show `⚠️ Open questions` here too. If §7 confirmed a real drift, show `⚠️ Drift` here too —
a ticket can carry any combination of the three.

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

- **One write only.** The refining-memory artifact under `.atl/memory/` (§10) is the ONLY file
  this skill writes. Do NOT write to any repo source file. Do NOT update Linear. Do NOT commit.
- **No guessing.** If a ticket is too vague to refine, flag `UNREFINED` and skip. Never invent scope.
- **No speculative abstractions.** Scope is what the ticket says. Don't expand it.
- **Contract over spec.** RESTClient `.http` and the trap list beat the extracted PDF docs.
- **Drift is surfaced, never inherited.** If a version/assumption disagrees with the manifest, flag it,
  never write a version-specific claim into the output that isn't traceable to §7's grep output, and
  make sure a confirmed drift lands in both the memory artifact and the §12 Flags column — not just
  in your reasoning.
- **Contradictions are surfaced, never silently resolved.** If ticket sections or comments disagree,
  that's an open question tagged with an addressee (→ Johnny / → Lean / → product), not a judgment
  call for this skill to make quietly.
- **Every comment gets closed out.** Incorporated, excluded, or open question — never dropped.
- **Escalation flags stop the handoff, not the refinement.** Mark them visibly — banner + table
  cell, not just prose. Architecture calls are Johnny's to make consciously; scope/product
  ambiguity gets tagged for Lean or product. This skill surfaces the decision; it doesn't make it.
- **The draft is audited against itself, not just against the ticket.** §5 checks the ticket's
  own sections for contradictions; §9's gate #6 checks the refinement's own draft (Scope vs
  Architecture decisions vs Estimate) for the same thing. Both matter — a clean ticket can still
  produce a self-contradicting refinement if the writing step isn't checked too.
- **The §9 gate is mandatory.** Do not write §10/§11 output until all six gate checks pass.
- **One ticket at a time.** Sequential, so context doesn't bleed.

---

**Language:** the refined description block matches the original ticket's language (see §11 table);
the Claudio prompt, the memory artifact, and the summary table stay EN regardless. Chat with Johnny
in Spanish.