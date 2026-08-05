## Review opened
- Timestamp: 2026-07-30
- Repo: FE
- Branch: feature/DEVSYM-522-pm-connection-health-check
- Base branch: stage
- Issue: DEVSYM-522

## Dev's stated focus
PM connection health-check during onboarding: new `GET /onboarding/pm/{pmConnectionId}/health-check`
wiring (api fn, query options, keys, types) and a "Locations & property access" card in the
PM onboarding wizard review step showing per-location RM property access (all / partial / none).

## Files in scope
src/api/index.ts
src/api/onboarding/keys.ts
src/api/onboarding/onboarding-pm-health-check.api.ts
src/api/onboarding/onboarding.ts
src/api/onboarding/queries/index.ts
src/api/onboarding/queries/pm-health-check.queries.ts
src/api/onboarding/types.ts
src/features/onboarding/PmOnboardingWizard.tsx
src/features/onboarding/pm-onboarding/steps/review-step.tsx

## Findings

### Blockers
None.

### Should-fix
None.

### Nits

**1. Severity color mismatch for `access === 'none'`**
File: `src/features/onboarding/pm-onboarding/steps/review-step.tsx`
Snippet:
```tsx
    case 'none':
      return (
        <span
          className={cn(
            ACCESS_PILL_BASE,
            'border-error-300 bg-error-100/60 text-error-700',
```
vs the advisory box for the same state:
```tsx
                  {location.access === 'none' ? (
                    <div className="mt-2 flex items-start gap-3 rounded-xl border border-amber-300 bg-amber-50 p-3">
```
The pill is red (`error-*`) but the box directly below is amber (warning). Same state, two
severity colors. Only NEW inconsistency this PR introduces. Suggest picking one severity.

**2. Duplicated pill-base + amber-notice chrome (systemic, optional)**
File: `src/features/onboarding/pm-onboarding/steps/review-step.tsx`
Snippet:
```tsx
const ACCESS_PILL_BASE =
  'inline-flex h-7 items-center rounded-full border px-3 font-body-md-regular text-body-sm leading-none'
```
```tsx
                    <div className="mt-2 flex items-start gap-3 rounded-xl border border-amber-300 bg-amber-50 p-3">
                      <AlertTriangle
                        className="size-4 shrink-0 translate-y-0.5 text-amber-500"
```
`ACCESS_PILL_BASE` is identical to `PmInviteStatusPill`'s `base` (4th copy repo-wide); the
amber box chrome is copied from `OnboardingBlockedNotice`. Both duplications are already
systemic in the repo — this PR FOLLOWS the existing convention, so it's an optional cleanup
(shared `InlineWarningNotice` + pill-base const), better as a separate PR.

**3. No Storybook coverage for new visual states**
File: `src/features/onboarding/pm-onboarding/steps/review-step.tsx`
Snippet:
```tsx
export function PmReviewStep({
  values,
  locations,
  isLoadingLocations,
}: {
```
New access branches (all/partial/none + loading) have no story; `review-step` had none before.
CONTRIBUTING mentions stories mirror components and `pnpm test` runs Storybook interaction tests.

## Notes / things verified (not findings)
- `ponytail:` comment prefix documenting the silent-omission-on-failure decision is a house
  convention (13 uses across 9 files) — NOT a leak/artifact.
- New `api.get<T>` fn + `queryOptions` calc the sibling conventions exactly (no Zod parse in
  onboarding siblings either; response interceptor already does camelCase).
- Double `enabled` (queryOptions `pmConnectionId > 0` + wizard `!!pmConnectionId && step === 2`)
  is harmless — wizard's `enabled` wins via spread order; queryOptions guard is good for reuse.
- Envelope `{ data: { locations } }` matches how the wizard reads `healthCheck.data?.data.locations`.
- Error/empty state intentionally omitted (documented in the ponytail comment) — conscious
  product decision, not an oversight.

## Verdict
ready to merge

## SPACE log
- Time spent reviewing: 20 min
- Model: Opus 4.8
- Findings: 0 blockers, 0 should-fix, 3 nits
- Verdict: ready to merge

## Review closed
- Timestamp: 2026-07-30
- Resolution: approved
- Notes: All findings were nits (0 blockers, 0 should-fix). Approved as-is; nits left as
  optional follow-ups (the InlineWarningNotice/pill-base extraction, if done, belongs in a
  separate cleanup PR — not scoped to 522).
- Johnny's review reading time: 20 min
