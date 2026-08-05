## Review opened
- Timestamp: 2026-07-30 22:09 -03
- Repo: BE
- Branch: `feature/devsym-522-pm-connection-health-check`
- Base branch: `dev`
- Issue: DEVSYM-522

## Dev's stated focus
Per-location RM property-access health-check for PM onboarding. New
`GET /api/onboarding/pm/{pmConnectionId}/health-check` endpoint that, for every
discovered RM location of the PM's corp, authenticates via the transversal
Partner Token and calls `GET Current/User?embeds=UserProperties`, classifying
per-location access as `all` (PropertyID -1 / star-all) / `partial` (explicit
list + property_count) / `none`. Informational, shown on the PM's own
onboarding review step; never blocks submission.

## Files in scope
```
app/Modules/Onboarding/Controllers/PmOnboardingController.php
app/Modules/Onboarding/Services/PmConnectionHealthCheckService.php
app/Modules/Onboarding/routes.php
docs/onboarding-pm-invite-activation.md
tests/Feature/Modules/Onboarding/PmOnboardingHealthCheckControllerTest.php
tests/Unit/Modules/Onboarding/Services/PmConnectionHealthCheckServiceTest.php
```

## Context note (not a finding)
`classifyAccess()` reads PascalCase RM fields (`UserProperties`, `PropertyID`)
in the SA layer. Normally a cardinal-rule violation, but the entire Onboarding
module does self-contained RM handshakes reading PascalCase directly
(`OnboardingActivationService`, `OnboardingService`, `OnboardingInviteService`).
Consistent with the documented module pattern → not flagged.

## Findings

### Should-fix

**File:** `app/Modules/Onboarding/Services/PmConnectionHealthCheckService.php`
Snippet:
```php
        try {
            $currentUser = $this->contextResolver
                ->clientForConnectionViaPartnerToken($connection)
                ->get('Current/User', ['embeds' => 'UserProperties']);
        } catch (\Throwable $e) {
            report($e);
```
- What's wrong: per-location isolation catch logs the full Throwable via `report($e)`.
- Why: RULES.md line 76 — RM fan-out per-item isolation must log sanitized
  (`exceptionClass = $e::class` + `exceptionMessage` only, NEVER full Throwable,
  RM bodies carry PII), mirroring `OnboardingService::initTasks()` (which has the
  explicit "avoid serializing the full Throwable which may include RM response
  bodies with PII" comment + `Log::warning` with class/message).
- Caveat: docstring cites `OnboardingInviteService` as precedent, and that service
  uses `report($e)` pervasively (lines 395/421/551/755/804/831) — repo is genuinely
  inconsistent. Documented rule + PII rationale favor the sanitized form.
- Suggested fix: replace `report($e)` with sanitized `Log::warning` including
  `location_id`/`rm_corpid`, `exceptionClass`, `exceptionMessage`.

### Nits

**File:** `app/Modules/Onboarding/Controllers/PmOnboardingController.php`
Snippet (identical block in `update()` and `healthCheck()`):
```php
        abort_if(
            $user->user_type !== 'sa_pm' || $user->pm_connection_id !== $pmConnectionId,
            403,
            'Not this user\'s PM connection.'
        );
```
- What's wrong: ownership guard duplicated verbatim across two methods.
- Why: code-review skill "no duplication"; two identical guards drift independently.
- Suggested fix: extract private `assertOwnsConnection(User $user, int $pmConnectionId): void`.

**File:** `tests/Feature/Modules/Onboarding/PmOnboardingHealthCheckControllerTest.php`
Snippet:
```php
    public function test_pm_cannot_check_another_connection(): void
    {
        $conn = $this->pmConnection();
        $other = $this->pmConnection(['rm_corpid' => 'othercorp']);
        $user = $this->pmUser($conn);

        $this->actingAs($user, 'sanctum')
            ->getJson("/api/onboarding/pm/{$other->id}/health-check")
            ->assertForbidden();
    }
```
- What's wrong: 403 test only covers the `pm_connection_id !== $pmConnectionId`
  branch; the `user_type !== 'sa_pm'` branch is untested.
- Why: guard is an OR of two conditions; a regression dropping the user_type
  check would still pass.
- Suggested fix: add a non-`sa_pm` user hitting their own connection id → assert 403.

## Verdict
ready to merge — no blockers. 1 should-fix (documented-rule logging alignment),
2 nits (polish). Safe to address in one small follow-up commit or waive with a note.

## SPACE log
- Time spent reviewing: TBD
- Model: Opus 4.8
- Findings: 0 blockers, 1 should-fix, 2 nits
- Verdict: ready to merge
