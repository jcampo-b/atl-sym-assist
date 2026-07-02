# SymAssist repo recon — 2026-07-02

Read-only reconnaissance pass. Structure and manifests only. No business logic read.

**Re-scanned on canonical branches:** FE `stage`, BE `dev`, AI `main`. Findings below reflect those branches. The earlier FE branch caveat (`DEVSYM-33`) is resolved.

## Repo names (confirmed on disk)

All three expected folders exist under `SYMASSIST_ROOT`; no name drift.

- BE: `SymAssist-Backend` ✓
- FE: `SymAssist-Frontend` ✓
- AI: `SymAssist-AI-Service` ✓

Other top-level dirs at root (context, not scanned): `Docs/`, `RestRequests/`, `Secrets/`, `symassist-docker/`.

## BE module ↔ RESTClient map

`RESTClient/` lives at backend root (`SymAssist-Backend/RESTClient/`), 14 `.http` files. `app/Modules/` has 13 modules. **The map you hold is CONFIRMED correct** — every `.http` belongs to a module, and every module maps.

| Module | RESTClient file | RM-backed? | Status |
|---|---|---|---|
| Tasks | tasks.http, tasks-counts.http, subtasks.http | yes | ✓ |
| TaskComment | task-comments.http | no | ✓ |
| TaskActivityLog | activity-log.http | no | ✓ |
| Units | units.http | yes | ✓ |
| Properties | (via integrations.http) | yes | ✓ no dedicated `.http` (as expected) |
| ContactInfo | contact-info.http | yes | ✓ |
| Onboarding | onboarding.http | yes | ✓ |
| UserProfile | user-profile.http | no | ✓ |
| Users | users-creation.http | no | ✓ |
| Auth | auth.http | dual (Sanctum + RM token) | ✓ |
| DailyPlanner | daily-planner.http | yes | ✓ |
| Dashboard | dashboard.http | yes | ✓ |
| Integrations | integrations.http | yes (RM boundary) | ✓ |

- **Modules with NO `.http`:** only `Properties` — and that is by design (covered via `integrations.http`).
- **`.http` files with NO module:** none. Every request file maps to a module.

**RM boundary path — CONFIRMED:** `app/Modules/Integrations/RentManager/`. Top-level contents (one level deep):
`Catalogs/`, `Controllers/`, `Mappers/`, `OpenApi/`, `Requests/`, `Services/`, `Traits/`, `routes.php`.
(Note: subdirs are `Catalogs` and `OpenApi` in addition to the expected `Services`/`Mappers`.)

## FE structure

- `src/features/` **exists**. Feature folders (8): `auth`, `dashboard`, `onboarding`, `settings`, `task-create`, `task-detail`, `task-search`, `workflow`.
- API-client layer **exists** at `src/api/` with 11 domains: `auth`, `chat`, `comments`, `dashboard`, `onboarding`, `properties`, `rent-manager`, `roles`, `task`, `tenants`, `users`.
- `src/api/dashboard/keys.ts` **EXISTS on `stage`** ✓ — matches your belief. (It was also present on the earlier `DEVSYM-33` feature branch, consistent with that branch having been cut from `stage`.)

## AI-Service (discovered)

- **Manifest of record:** `pyproject.toml` — it is the **only** dependency manifest present. No `requirements.txt`, `poetry.lock`, `uv.lock`, or `Pipfile`. So there is no ambiguity: `pyproject.toml` is authoritative. (Which resolver CI uses is not determinable from manifests alone — a `Makefile`, `Dockerfile`, and `.pre-commit-config.yaml` exist but were not opened this pass.)
- **Python:** `requires-python = ">=3.12"` (mypy pinned `python_version = "3.12"`).
- **FastAPI:** `fastapi>=0.115.0`. Also declares `python-dotenv>=1.0.0`, `sentry-sdk[fastapi]>=2.20.0`.
- **Endpoint organisation:** package layout `src/app/` with entrypoint `src/app/main.py`. Routers under `src/app/api/v1/` — `chat.py` and `health.py` (plus `__init__.py`). Supporting packages: `core/`, `models/` (+ `models/db/`), `services/` (+ `services/backend/`, `services/tools/`). Alembic migrations at `src/alembic/`. So: **versioned routers, not a single monolithic main.py.**
- **RESTClient equivalent — FOUND:** a Postman collection at `postman/`:
  `SymAssist-AI-Service.postman_collection.json` + `SymAssist-AI-Service.postman_environment.json`.
  Additionally `tests/` has `api/`, `core/`, `services/` subdirs (test-based coverage, not request fixtures). So the AI repo's request contract lives in Postman, not in a `.http` dir like BE.

## Drift found

| Item | Your belief | On disk | Verdict |
|---|---|---|---|
| BE PHP | 8.2 | `"php": "^8.2"` | ✓ match |
| BE Laravel | 12 | `"laravel/framework": "^12.0"` | ✓ match |
| FE React | 19 | `"react": "^19.2.0"` | ✓ match |
| AI FastAPI | unknown | `>=0.115.0` | ✓ resolved (no prior belief) |
| AI Python | unknown | `>=3.12` | ✓ resolved (no prior belief) |
| FE `dashboard/keys.ts` | stage-only | present on `stage` | ✓ match (re-scanned) |

**No drift found.** All declared dependencies match your beliefs, and the `dashboard/keys.ts` assumption is confirmed on `stage`.

Bonus FE versions observed (no prior belief): `@tanstack/react-query ^5.90.21`, `zod ^4.3.6`, `typescript ~5.9.3`.

## .atl wiring

- `.atl/docs/extracted/` — 4 `.md` present ✓:
  `01-Rent-Manager-API-Overview.md`, `02-Rent-Manager-API-QuickStart.md`, `03-Partner-Network-Resource-Guide-2024.md`, `04-Voice-AI-V3-Planning.md`.
  ⚠️ `.DS_Store` present (6148 bytes) — should be removed / gitignored.
- `.atl/memory/RULES.md` — OK ✓
- `.atl/linter.md` — OK ✓

## Open questions for Johnny

1. ~~FE branch~~ — RESOLVED. Re-scanned on `stage`; feature list, `src/api`, and `dashboard/keys.ts` all confirmed. BE re-scanned on `dev` (module ↔ `.http` map and RM boundary identical). AI on `main`.
2. **AI CI resolver:** `pyproject.toml` is the sole manifest, but I did not open `Makefile` / `Dockerfile` / `.github/` to confirm whether CI installs via `pip`, `uv`, or `poetry`. Not determinable from manifest presence alone — flag as "not verified" until those files are read.
3. **AI endpoint surface:** only `chat` and `health` routers exist under `api/v1`. If you expected more endpoints, either they are not built yet or live outside `api/v1` (not found this pass).
