# DEVSYM-373 — [BE] Superuser roles + cross-PM access policy + fan-out
> Refined 2026-07-09, grounded in the read-only repo recon of `dev`
> (`refinement/BE/2026-07/2026-07-09-devsym-373-role-policy-fanout-recon.md` — evidence source).
> Source of truth is still the Linear ticket + AGENTS.md.

## Current ask (as read from Linear)
Define the superuser roles and let a superuser query fan out across all `active` PM connections a
role reaches, aggregating results. Sits on the DEVSYM-399 credential foundation. Out of scope on the
ticket: data aggregation/caching (DEVSYM-374). The body is written loosely ("assignable roles",
"fans out and aggregates") without naming the 373/374 seam or the role-storage owner.

## Target
Repo(s): BE • Module: NEW SA domain machinery (mirror `app/Modules/Tasks/StateMachine/`) • Layer(s): SA domain •
RM-backed: yes, via the DEVSYM-399 foundation (`RentManagerContextResolver::clientForConnection`).
Contract source: repo recon (see evidence doc) + DEVSYM-399 as-built + DEVSYM-392 rate-limit findings.

## Refined scope
Pure stateless domain machinery — NO table, NO endpoint, NO auth:
- `enum Role` — SA source of truth for the five superuser roles (backed enum, `match`-based methods,
  RM mapping kept OUT — mirrors `TaskStatus`).
- `active` scope on `PmConnection` (`scopeActive` → `where('status', 'active')`) — does not exist yet.
- `SuperuserAccessPolicy::reachableConnections(Role): Collection<PmConnection>` — decides which `active`
  connections a role reaches (credential-mode-agnostic; operates on the model, not on a logged-in user).
- `CrossPmFanout::fanOut(Role, callable): array<int rm_location_id, array>` — resolves the reachable
  connections, builds one foundation-scoped client per connection via `clientForConnection`, runs the
  caller-supplied operation, and collects results keyed by `rm_location_id`.
- Unit tests only (stateless: role value in → connections/clients/results out).

NOT in 373: caching / counts / PM-tagging / HTTP endpoint (DEVSYM-374); `roles` table / `internal_users`
/ `role_id` (DEVSYM-376); credentials + `clientForConnection` (already built in DEVSYM-399); login/session
(later auth ticket).

## Comment ledger
No comments on the ticket. Nothing to close out.

## Contradictions / open questions
NONE. The one prior ambiguity — "assignable role" implying 373 needs a store — is RESOLVED:
**373 DEFINES the roles (`enum Role`); DEVSYM-376 ASSIGNS them** (`roles` table seeded from `Role::cases()`,
`internal_users`, `role_id`). The 373→374 seam is also settled (see "Contract produced for 374").

## Architecture decisions that apply
- **Credential-mode-agnostic.** 373 reaches connections and hands each to `clientForConnection`; whether a
  connection authenticates via user-session or partner-token is a `PmConnectionContext::credentialMode()`
  branch (DEVSYM-399), invisible to 373.
- **DI direction: 376 → 373.** `Role` is the source of truth; DEVSYM-376's `roles` table seeds FROM
  `Role::cases()`. 373 must NOT depend on 376 (no table, no `internal_users`), keeping it unit-testable
  standalone.
- **Sequential fan-out, no pool.** RM rate limits are isolated per customer DB (DEVSYM-392); one request
  per Location in a single pass is within budget. Do NOT stagger and do NOT use `poolGet`/`pool` (YAGNI).
- **Mirror `Tasks/StateMachine/`**, not the `Users/` CRUD skeleton — this is table-less, endpoint-less
  domain machinery.

## Contract produced for 374
`CrossPmFanout::fanOut(Role, callable)` returns **`array<int rm_location_id, array>`** — one raw
decoded-JSON array per reached connection (`RentManagerHttpClient::get()` returns `array`). 374 supplies
the per-connection operation, caches per `rm_location_id`, then merges/tags/counts. **373 never pre-merges**
— the per-Location split is what makes 374's per-Location caching possible.

## Traps in play
- `PmConnection` has NO scopes/relations today — 373 must ADD the `active` scope from scratch.
- NEVER pre-merge results in 373 (would collapse the Location keys 374 caches on).
- 373 is the FIRST caller of `clientForConnection` (zero callers today — DEVSYM-399 scaffolding). Before
  wiring a real caller through `PmConnectionContext`, re-check `EnsureRmTokenValid`'s expiry gate (validated
  only against `UserSessionContext` — see RULES.md / DEVSYM-404).
- Do NOT touch the `EnsureIsSuperAdmin` gate, the `users` table, or credential storage — all owned elsewhere.
- `Role` ≠ RM `rm_roles`. `RoleService`/`RoleController` (Users module) are RM-remote passthroughs, not an
  SA store — do not route `Role` through them.
- Sequential fan-out only — no staggering, no pooled concurrency.
- Bash output in this repo is redaction-mangled; use the Read tool for exact signatures.

## Drift / flags
- `Role` is greenfield — no SA-owned role enum/table/model exists today (only the RM-passthrough
  `RoleService` and the unrelated `TaskStatus` enum). Creating `enum Role` overlaps with nothing.
- Estimate corrected DOWN: the roles-table / account-management / assignment work that inflated earlier
  reads is DEVSYM-376's, not 373's. 373 is enum + scope + policy + fan-out service + unit tests.

## Suggested estimate
**3 pts.** Enum + `active` scope + `SuperuserAccessPolicy::reachableConnections` +
`CrossPmFanout::fanOut` (sequential, keyed by `rm_location_id`) + unit tests, on top of the DEVSYM-399
foundation. (Down from a naive 5 once roles-table/account work is correctly attributed to DEVSYM-376.)
