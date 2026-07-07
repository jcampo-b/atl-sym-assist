## Review opened
- Timestamp: 2026-07-03 16:35 -03
- Repo: FE
- Branch: DEVSYM-185-fe-integrate-owner-onboarding-with-backend-api
- Base branch: stage
- Issue: DEVSYM-185

## Dev's stated focus
Integrate owner onboarding with the backend API.

## Files in scope
src/api/auth/schemas.test.ts
src/api/auth/schemas.ts
src/api/auth/types.ts
src/api/index.ts
src/api/onboarding/invite.mappers.test.ts
src/api/onboarding/invite.mappers.ts
src/api/onboarding/keys.ts
src/api/onboarding/mappers.test.ts
src/api/onboarding/mappers.ts
src/api/onboarding/onboarding-prefill.api.ts
src/api/onboarding/onboarding.ts
src/api/onboarding/queries/index.ts
src/api/onboarding/queries/onboarding-prefill.queries.ts
src/api/onboarding/schemas.ts
src/api/onboarding/types.ts
src/components/wizard/WizardModalShell.tsx
src/features/auth/LoginPage.tsx
src/features/dashboard/components/DashboardOnboardingModals.tsx
src/features/onboarding/TenantOnboardingWizard.tsx
src/features/onboarding/UserOnboardingWizard.tsx
src/features/onboarding/VendorOnboardingWizard.tsx
src/features/onboarding/user-onboarding/steps/owner-profile-step.tsx
src/features/onboarding/user-onboarding/steps/review-invite-step.tsx
src/features/onboarding/user-onboarding/types.ts
src/features/settings/users/AddUserPermissionsStep.tsx
src/features/settings/users/AddUserWizard.tsx
src/features/settings/users/sa-onboarding-roles.ts
src/routes/_authenticated/tenant-onboarding/-TenantOnboardingPage.tsx
src/routes/_authenticated/user-onboarding/-UserOnboardingPage.tsx
src/routes/_authenticated/vendor-onboarding/-VendorOnboardingPage.tsx

## Overview
Large, clean refactor (+516 / −1230). Removes the entire onboarding `prefill` flow
(queries/mappers/schemas/api/tests), replaces role-ID-vs-roles-API selection with a
static `SA_ROLES` list, drops password creation from Add User in favor of the invite
flow, splits `fullName` into `firstName`/`lastName`, and adds a `missingFields`
highlight path in the owner wizard. Login page gets a show/hide password toggle.
Wizard shells get a `disabled` prop to lock nav while submitting.

## Findings

### should-fix (confirm intended)

**File:** src/features/settings/users/AddUserWizard.tsx
```tsx
    if (step === 3) {
      if (requiresSingleProperty) return values.propertyIds.length === 1
      return values.propertyIds.length > 0
    }
```
All SA roles now require >=1 property to advance step 3, including the new
`sa_vendor_emp`. The old `!requiresProperties` branch (2->4 skip) was removed
entirely. If the backend allows inviting a vendor employee without a property,
this blocks a valid case. Confirm against the `/onboarding/invite` contract.

**File:** src/api/onboarding/mappers.ts
```ts
export function entityNeedsAttention(entity: OnboardingEntitySummary | null): boolean {
  if (!entity || entity.id <= 0) return false
  return entity.missing.length > 0
}
```
Now ignores `needsLinking` and `tag` (pending/in_progress); only checks
`missing.length`. Tests were updated to match, so it looks deliberate — but a
`needsLinking: true` entity with no missing fields no longer flags attention,
which could hide a broken RM link. Confirm the narrowing is intended.

### nits

**File:** src/features/settings/users/sa-onboarding-roles.ts
```ts
// ponytail: static role list per product decision — no roles API call in Add User
export const SA_ROLES: SaRole[] = [
```
Stray `ponytail:` marker in the comment (codename / AI tag?). The rationale is
fine; drop the prefix.

**File:** src/api/onboarding/types.ts
```ts
export interface OnboardingOwnerPatchBody {
  firstName?: string
  lastName?: string
  companyName?: string | null
  isCompany?: boolean
  email?: string
  taxId?: string | null
  comment?: string | null
  isActive?: boolean
  primaryContact?: OnboardingPrimaryContact
  addresses?: OnboardingAddress[]
  phoneNumbers?: Array<{ number: string }>
  contacts?: unknown[]
  propertyIds?: number[]
  userDefinedValues?: unknown[]
}
```
`taxId`, `comment`, `addresses`, `contacts`, `userDefinedValues` are declared but
the only producer in the diff (`buildOwnerPatchBody`) sets none of them; it only
adds `isActive`. `contacts`/`userDefinedValues` are `unknown[]`. Speculative type
surface + weak typing on the wire contract (code-review skill: build only what the
ticket asks). Trim to what we send today, type the rest when wired.

## Verdict
ready to merge — no blockers. The two should-fix items are behavior confirmations,
not verified defects; nits are cosmetic. Merges as-is if vendor_emp-without-property
and the needsLinking drop are intentional.

## SPACE log
- Time spent reviewing: 15 min
- Model: Opus 4.8
- Findings: 0 blockers, 2 should-fix, 2 nits
- Verdict: ready to merge (pending 2 confirmations)
