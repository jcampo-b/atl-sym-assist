# PR Review — DEVSYM-395

## Review opened
- **Date:** 2026-06-30
- **Repo:** BE
- **Branch:** `feature/DEVSYM-395-onboarding-rework`
- **Base branch:** `dev`
- **Issue:** DEVSYM-395

## Dev's stated focus
Onboarding rework — list endpoints with status tags (`/owners`, `/vendors`, `/tenants`), `/me` endpoint for the logged-in user's own onboarding status, step-based completeness engine (`OwnerOnboardingSteps`, `TenantOnboardingSteps`, `VendorOnboardingSteps`), `RentManagerFieldPresence` extractor, user-to-RM-entity linking (`POST /users/{userId}/link`).

---

## Findings

### 🔴 Blockers

**1. Base migration modified**
- **File:** `database/migrations/0001_01_01_000000_create_users_table.php:29–33`
- **What:** Five new columns (`last_login_at`, `linked_owner_id`, `linked_tenant_id`, `linked_vendor_id`, `linked_vendor_emp_id`) added to the initial users migration instead of a new timestamped migration.
- **Why:** RULES.md explicit rule — "Never add new columns to a base migration." Staging and prod already ran this migration — those environments silently miss all five columns.
- **Fix:** Create `2026_06_30_000001_add_onboarding_link_fields_to_users_table.php` with `up()`/`down()`. Remove those lines from the base migration.

---

### 🟡 Should-fix

**2. `UserController::link` reads raw from `$request->validated()`**
- **File:** `app/Modules/Users/Controllers/UserController.php:29–30`
- **What:** Uses `$validated['entity_type']` and `(int) $validated['entity_id']` instead of FormRequest helpers.
- **Why:** AGENTS.md — "Always use FormRequest helpers (`boolean()`, `integer()`, `string()`) instead of reading raw from `$this->validated()`."
- **Fix:** Use `$request->string('entity_type')` and `$request->integer('entity_id')` directly.

**3. `TenantOnboardingSteps::rmTag` / `VendorOnboardingSteps::rmTag` use a different field set than their `evaluate()`**
- **Files:** `app/Modules/Onboarding/Steps/TenantOnboardingSteps.php:42` · `VendorOnboardingSteps.php:55`
- **What:** `rmTag()` delegates to the mapper's `detectMissingFields()` (mapper's REQUIRED_FIELDS) while `evaluate()` iterates the class's own STEPS. `OwnerOnboardingSteps` avoids this with its own `isComplete()` that walks STEPS.
- **Why:** If the two field sets diverge, list `tag` and per-step `evaluate()` can contradict each other.
- **Fix:** Add a private `isComplete()` in both classes that iterates `self::STEPS` via `RentManagerFieldPresence::detectMissing()`, replacing the mapper delegation in `rmTag()`.

**4. SA layer reads PascalCase RM field names directly**
- **File:** `app/Modules/Onboarding/Services/OnboardingService.php:369/396/423`
- **What:** `$rrmRaw['Status']`, `$rrmRaw['IsProspect']`, `$rrmRaw['IsActive']` accessed in SA service.
- **Why:** AGENTS.md cardinal rule — SA layer never reads PascalCase RM field names.
- **Fix:** Add `lifecycle/status`, `is_prospect`, `is_active` to `toOwner/toTenant/toVendor` mapper methods and read from the mapped result.

**5. No feature test for `POST /users/{userId}/link`**
- **File:** `tests/Feature/Modules/Users/` (missing)
- **What:** Endpoint has no feature test — happy path, 404, 403, 422 all untested.
- **Why:** RULES.md — "Every new or modified endpoint must have a feature test asserting the response shape."
- **Fix:** Add `LinkUserControllerTest` with: happy path (assert correct linked_*_id written to DB), 404 on unknown rm_user_id, 403 for non-admin, 422 for invalid entity_type.

---

### ⚪ Nits

**6. `OwnerOnboardingSteps` doesn't implement `OnboardingStepsContract`**
- The other two step classes implement it; OwnerOnboardingSteps doesn't — missing `steps()` and `evaluate()`.
- Fix: Either implement the contract uniformly, or drop it from the other two.

**7. Unreachable `default` branch in `UserService::link`**
- `default => throw new \InvalidArgumentException` can never fire — entity_type is validated upstream.
- Fix: Remove the default branch.

**8. Spanish comment in `routes.php:950`**
- `// Tags funcionales una vez que PR 5 linkee SA users...` — code comments should be in English.
- Fix: Translate or move to PR description.

**9. `.env.example` — missing EOF newline + unexplained empty var**
- No trailing newline after `SENTRY_TRACES_SAMPLE_RATE=1.0`.
- `RM_SA_VENDOR_EMP_ROLE_ID=` has no inline comment.
- Fix: Add trailing newline; add `# Leave empty until vendor employee role is provisioned in RM`.

---

## Verdict
**Needs changes before merge.**
Blocker: base migration. Should-fix before merge: FormRequest helpers, rmTag/evaluate inconsistency, PascalCase-in-SA, missing link test.

---

## SPACE log

```
PR review — DEVSYM-395 (BE)
Time spent reviewing: TBD
Model: Sonnet (scoped feature review)
Findings: 1 blocker, 4 should-fix, 4 nits.
Verdict: needs changes before merge
```
