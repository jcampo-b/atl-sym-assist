## Review opened
- Timestamp: 2026-07-08
- Repo: FE
- Branch: `DEVSYM-188-fe-integrate-vendor-onboarding-with-backend-api`
- Base branch: `stage`
- Issue: DEVSYM-188

## Dev's stated focus
Integrate vendor onboarding with the backend API.

## Files in scope
```
src/api/onboarding/mappers.test.ts
src/api/onboarding/mappers.ts
src/api/onboarding/schemas.ts
src/api/onboarding/types.ts
src/api/rent-manager/keys.ts
src/api/rent-manager/queries/service-categories.queries.ts
src/api/rent-manager/service-categories.api.ts
src/api/rent-manager/types.ts
src/components/wizard/primitives.tsx
src/features/dashboard/components/DashboardOnboardingModals.tsx
src/features/onboarding/TenantOnboardingWizard.tsx
src/features/onboarding/VendorOnboardingWizard.tsx
src/features/onboarding/onboarding-routes.ts
src/features/onboarding/shared/service-scope-step.tsx
src/features/onboarding/shared/wizard-step-footer.tsx
src/features/onboarding/vendor-employee-onboarding/steps/vendor-profile-step.tsx
src/features/onboarding/vendor-onboarding/steps/admin-profile-step.tsx
src/features/onboarding/vendor-onboarding/steps/review-invite-step.tsx
src/features/onboarding/vendor-onboarding/steps/vendor-profile-step.tsx
src/features/onboarding/vendor-onboarding/types.ts
src/features/settings/users/AddUserWizard.tsx
src/features/settings/users/sa-onboarding-roles.ts
src/routes/_authenticated/vendor-onboarding/-VendorOnboardingPage.tsx
src/stories/components/wizard/wizard-primitives.stories.tsx
```

## Findings

### should-fix

**File:** `src/features/onboarding/vendor-onboarding/steps/vendor-profile-step.tsx`
```tsx
import { MOCK_SERVICE_CATEGORIES } from '@/features/onboarding/shared/constants'
...
          {openSelect === 'serviceCategory' ? (
            <div className="absolute z-10 mt-2 w-full rounded-xl border border-gray-200 bg-white p-1 shadow-overlay">
              {MOCK_SERVICE_CATEGORIES.map((opt) => (
```
What's wrong: primary "Service category*" dropdown (step 1) still hardcoded from `MOCK_SERVICE_CATEGORIES`, while step 2 ("Service scope") was rewired in this same PR to fetch live categories via `rentManagerServiceCategoriesQueryOptions()`.
Why: inconsistent with the PR's own goal (real backend/RM integration); a vendor can pick a category that doesn't exist in RM for that tenant.
Suggested fix: pass the same `serviceCategories` query result into `VendorProfileStep` and render options from it instead of the mock constant.

### nit

**File:** `src/features/onboarding/vendor-onboarding/types.ts` / `review-invite-step.tsx`
```ts
export const VENDOR_ONBOARDING_STEPS = [
  { n: 1, label: 'User profile' },
```
```tsx
<h4 className="font-body-md-semibold text-gray-800">Vendor profile</h4>
```
What's wrong: stepper label for step 1 renamed to "User profile" but the step heading and review summary still say "Vendor profile".
Why: inconsistent copy for the same step.
Suggested fix: align wording across stepper, step heading, and review summary.

## Verdict
needs changes before merge

## SPACE log
- Time spent reviewing: 30 min
- Model: Sonnet
- Findings: 0 blockers, 1 should-fix, 1 nit
- Verdict: needs changes before merge
