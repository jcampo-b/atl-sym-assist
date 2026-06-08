# SymAssist — Agent Instructions

## Project
SymAssist is a property-management assistant backed by a Laravel 12 modular monolith (`SymAssist-Backend/`, PHP 8.3+). The backend acts as a BFF over the RentManager (RRM) property-management API: it authenticates against RentManager, proxies and reshapes RRM data into stable snake_case contracts, and exposes task management, daily planning, dashboard KPIs, onboarding, and contact-info endpoints for the frontend. RentManager remains the system of record; the backend owns translation, caching, and the API surface the FE consumes.

## Source of truth
Read before any task:
- `SymAssist-Backend/docs/ARCHITECTURE.md` — system overview
- `SymAssist-Backend/docs/rent-manager-integration.md` — RM integration mechanics
- `SymAssist-Backend/docs/rent-manager-filters.md` — RM filter string reference
- `.atl/AGENTS.md` (this file) — conventions and routing

## Active modules
All paths are under `SymAssist-Backend/app/Modules/`.

| Module | Layer | Owner agent | RESTClient file |
|---|---|---|---|
| Auth | SA (Sanctum + RM token) | rm-integration | `RESTClient/auth.http` |
| ContactInfo | SA proxy → RM | rm-integration | `RESTClient/contact-info.http` |
| DailyPlanner | SA proxy → RM | task-filters | `RESTClient/daily-planner.http` |
| Dashboard | SA proxy → RM | rm-integration | `RESTClient/dashboard.http` |
| Integrations/RentManager | RM | rm-integration | `RESTClient/integrations.http` |
| Onboarding | SA (Owner/Tenant/Vendor/VendorEmployee) | new-module | — |
| Properties | SA proxy → RM | rm-integration | — |
| TaskActivityLog | SA proxy → RM | rm-integration | `RESTClient/activity-log.http` |
| TaskComment | SA proxy → RM | rm-integration | `RESTClient/task-comments.http` |
| Tasks | SA proxy → RM | task-filters | `RESTClient/tasks.http`, `RESTClient/tasks-counts.http`, `RESTClient/subtasks.http` |
| Units | SA proxy → RM | rm-integration | — |
| UserProfile | SA (local DB) | new-module | — |

## Layer architecture — the cardinal rule
Data flows in one direction through four layers:

```
RRM (RentManager API)  →  RM (Integrations/RentManager)  →  SA (other Modules)  →  FE (frontend)
```

- **RRM** speaks PascalCase (`ServiceManagerIssue`, `StatusID`, `DueDate`).
- **RM layer** (`app/Modules/Integrations/RentManager/`) is the ONLY place PascalCase RM field names are allowed. It translates RM ⇄ SA.
- **SA layer** (all other modules) speaks snake_case and never knows RM field names.
- **FE** consumes only SA snake_case contracts.

If you see PascalCase field names outside `app/Modules/Integrations/RentManager/` → STOP. That logic belongs in the RM layer.

## Naming conventions

| Context | Convention | Example |
|---|---|---|
| SA fields (JSON, DTOs, DB) | snake_case | `due_date`, `status_id`, `task_id` |
| RM fields (RM layer only) | PascalCase | `DueDate`, `StatusID`, `ServiceManagerIssueID` |
| PHP classes | PascalCase | `RentManagerIssueMapper`, `TaskService` |
| DB tables | snake_case plural | `user_profiles`, `task_comments` |
| Route names | dot notation | `tasks.index`, `integrations.rm.issues.show` |
| DTOs | PascalCase + `Data` suffix | `TaskData`, `TaskListData` |

## API conventions
- **BFF rule**: the FE talks only to SA endpoints (`/api/...`). It never calls RRM directly.
- SA endpoints: `/api/{resource}` (e.g. `/api/tasks`, `/api/daily-planner`).
- RM passthrough/integration endpoints: `/api/integrations/rent-manager/{resource}`.
- HTTP verbs: `GET` list/show, `POST` create, `PUT`/`PATCH` update, `DELETE` remove.
- Response format: always `JsonResponse`. List responses carry pagination meta from the RM response (`meta.pagination.total`).

## RentManager integration — quick reference
- Base URL: `https://cpmomaha.api.rentmanager.com/`
- Token: cached in Redis ~23h via `RentManagerAuthService`. Never call the RRM auth endpoint directly from any other class.
- Dev bypass: `DEV_AUTH_BYPASS=true` skips Sanctum (via `BypassAuthForDev` middleware). The RM token is still required.
- Filters string: `"Field,op,value;Field2,op2,value2"` — semicolon-separated, AND-only, no OR operator.
- embeds: always include `Status` for ServiceManagerIssues.
- RM field names: check `docs/rent-manager/raw/` (e.g. `00-models.json`, `embeds-*.json`) before guessing.
- All HTTP to RRM goes through `RentManagerHttpClient` only.

## Known traps
Non-obvious constraints discovered in the codebase. All verified against source.

- **No OR operator.** RM filters are AND-only. `schedule[]` accepts only ONE value per request (`RentManagerIssueMapper::buildScheduleClauses()`).
- **`STATUS_PENDING = 39`** is a hardcoded RM status ID (`RentManagerIssueMapper::STATUS_PENDING`). If this ID varies across environments, move it to config — today it is a class constant.
- **`StatusID` is not returned at the top level** on ServiceManagerIssues. It only comes back via the embedded `Status` relation as `ServiceManagerStatusID`. Always include `Status` in embeds (`RentManagerIssueMapper::ALWAYS_EMBED = 'Status'`), or status will be null.
- **RM returns deleted units in the active list** — filter them out in the RM layer, not in SA.
- **`orderingOptions` returns 404** from RRM for ServiceManagerIssues in this tenant instance. Do not rely on it for task sorting.
- **`TaskService` never calls RRM directly** — always via `RentManagerService` + `RentManagerIssueMapper`. If you find yourself injecting `RentManagerHttpClient` from a non-Integrations service → wrong.
- **Chip counts will not sum to the total.** Schedule buckets (overdue/scheduled/pending/completed) intentionally overlap.
- **RM assigns `ServiceManagerStatusID: 1` (`<Unassigned>`) to newly created issues with no explicit status.** This ID is not a named lifecycle status but must be handled — it maps to `TaskStatus::Virgin` in `RM_TO_LIFECYCLE` (read path only). It is intentionally absent from `LIFECYCLE_TO_RM`; any write uses ID 56.
- **RM rejects issue updates without Title.** Any PATCH to `ServiceManagerIssues` must include `Title`. Omitting it returns `"Issue cannot be blank."`. Always carry `Title` from the fetched raw issue (`$raw['Title'] ?? ''`) in any `updateServiceManagerIssue` call.

## Skills
Read the relevant skill before implementing:
- New RM resource or mapper change → `.atl/skills/rm-integration/SKILL.md`
- New Laravel module → `.atl/skills/new-module/SKILL.md`
- New task filter or schedule bucket → `.atl/skills/task-filters/SKILL.md`
- Any git operation (branch, PR, commit) → `.atl/skills/git-workflow/SKILL.md`
- Before committing any implementation → `.atl/skills/code-review/SKILL.md`

## Boundaries — read this first
- `.atl/` is the agent layer. All agent files (`AGENTS.md`, `skills/`, `memory/`) live here.
- `SymAssist-Backend/` is the actual codebase. All real work happens there.

The agent layer NEVER modifies itself during work — it only reads from it. Every task means: read `.atl/` for context and rules, then go work inside `SymAssist-Backend/`.

- Never create application files inside `.atl/`.
- Never create agent/config files inside `SymAssist-Backend/`.

## Environment rules
- Do NOT install any tools, packages, or binaries on the host machine.
- Use only what is already available. If a tool is missing, work without it.
- Do not narrate your environment or setup. Just do the task.

## Code quality rules
- No over-engineering. Build what's needed now.
- Reuse before creating. Check existing services/mappers first.
- Comments only when the WHY is non-obvious.
- Controllers thin. Logic in Services. Shape in DTOs.
- No magic constants with semantic meaning — name them.
- No speculative abstractions. Before creating an interface or abstract class,
  confirm there is more than one concrete implementation today, or a ticket that
  requires one. If neither → inline it.
- Before adding a method to a mapper or service, confirm it is called from more
  than one place. If it's a one-off payload builder → build it inline at the call site.
- Do not create stub files or methods with no callers. Empty DTOs, no-op methods,
  and stub rules that return false belong in their own ticket when the logic is known.
  Shipping them as placeholders adds noise without value.
- `TaskStatus::cases()` returns enum cases in declaration order — no sort needed.
  Never add `usort` on top of it.
- The rule engine fires AFTER a transition is written. Logic that bypasses
  `assertCanAdvance()` (emergency, rework) goes in `TaskService::updateStatus()`,
  not in a rule.

## Laravel 12 rules
- Providers: `bootstrap/providers.php` only.
- Middleware: `bootstrap/app.php` only (no `Http/Kernel.php`).
- Scheduler: `routes/console.php` (no `Console/Kernel.php`).
- All `.md` docs → `/docs` (except `README.md` in root).
- PHP 8.3+: use constructor promotion, `match`, enums, `readonly`.

## Git workflow
Read `.atl/skills/git-workflow/SKILL.md` before any git operation.

### Branch — do this before touching any code

**Step 1 — check if a branch for this ticket already exists:**
```bash
git fetch origin
git branch -a | grep devsym-<ticket_number>
```

**Step 2 — decide:**

- **Branch exists AND its type matches the current task** (e.g. existing `feat/devsym-319-...`
  and we're doing feature work) → check it out and use it.
  ```bash
  git checkout feat/devsym-319-status-transition-engine
  ```

- **Branch exists BUT its type doesn't match the current task** (e.g. existing
  `feat/devsym-319-...` but we're now fixing a bug on that ticket) → create a new branch
  with the correct type. Both can coexist; the type prefix reflects the nature of the work.
  ```bash
  git checkout dev
  git pull origin dev
  git checkout -b fix/devsym-319-status-transition-edge-case
  ```

- **No branch exists for this ticket** → create from `dev`:
  ```bash
  git checkout dev
  git pull origin dev
  git checkout -b <type>/devsym-<ticket_number>-<ticket-title>
  ```

Branch naming:
- `<type>`: `feat` for new functionality, `fix` for bug fixes.
- `<ticket_number>`: the Linear ticket number (e.g. `319`).
- `<ticket-title>`: ticket title, lowercased and hyphenated, kept readable.
- Examples:
  ```
  feat/devsym-319-status-transition-engine
  feat/devsym-149-autoapproval-threshold
  fix/devsym-319-status-transition-edge-case
  fix/devsym-184-exception-auth-roles
  ```
- Never branch from another `feat/*` or `fix/*` branch. Always from `dev`.

A PR that changes any API surface MUST include a Testing Endpoints section.

## Human review gate
Before committing or opening a PR, stop and notify the user:

"Implementation is ready. Please review the code in your IDE before I commit.
Let me know when to proceed."

Do NOT commit or push until the user explicitly confirms.

## Commit discipline
- One logical change per commit. A ticket with multiple files does not mean one commit.
- Split by concern: core logic / wiring / tests are separate commits within the same branch.
- Small, reviewable diffs are non-negotiable. If a single commit touches more than ~8 files,
  stop and ask whether it should be split.
- Never bundle unrelated changes in the same commit.

## Commit authorship
Never add `Co-Authored-By` or any attribution to Claude in commit messages.
Never add any authorship at all.

## Before opening a PR
Sync with dev to ensure no conflicts before opening the PR:

```bash
git fetch origin
git merge origin/dev
```

If there are conflicts, resolve them, run `docker compose exec app php artisan test`, and confirm all
pre-existing failing tests were already failing on `dev` before proceeding.

## Memory
At the start of every new session, read all files in `.atl/memory/` before
doing any work. These files contain corrections, known traps, and decisions
from prior sessions that must not be repeated.

After completing any task, write an entry to `.atl/memory/`:
- Filename: `YYYY-MM-DD_HH-MM_<short-slug>.md`
- Sections: What was done / Files touched / Decisions made / Open questions

After receiving a correction or code review finding, write an entry to `.atl/memory/`:
- Filename: `YYYY-MM-DD_HH-MM_correction-<short-slug>.md`
- Sections: What was wrong / What the correct approach is / Rule to apply going forward

After receiving PR review comments, save each finding to memory before
applying any fix.

This ensures corrections are not repeated across tickets.