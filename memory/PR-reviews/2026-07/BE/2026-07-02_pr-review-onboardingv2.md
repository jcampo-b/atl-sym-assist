## Review opened
- Timestamp: 2026-07-02
- Repo: BE
- Branch: bugfixing/onboardingv2
- Base branch: dev
- Issue: onboardingv2 (no ticket number in branch name)

## Dev's stated focus
Owner onboarding data (name, email, phone, address) wasn't saving — fields were written to the wrong
place on the RRM Owner record. `/me` and `/prefill` overlapped, and the "missing fields" check read the
wrong source, flagging everything as missing even when complete.

Fix: owner write-path now targets where RRM actually stores data — name/email on the related Contact,
phone under the Contact, address under the Owner, written via sub-resource endpoints. `/me` is now the
single onboarding-status endpoint (`/prefill` removed), returns a flat `missing` list instead of nested
"steps", and required fields are declared once per `user_type` via new `OnboardingDefinition` classes.

Known follow-ups the dev already flagged as out of scope:
- Object-level auth on write endpoints (IDOR) — already tracked in memory as `project_onboarding_idor_pending`.
- Live-verify `/me` for tenant and vendor (owner verified end-to-end).
- lifecycle→tag mapping (Future caps at pending).

## Files in scope
```
RESTClient/onboarding.http
app/Modules/Integrations/RentManager/Mappers/RentManagerOwnerMapper.php
app/Modules/Integrations/RentManager/Mappers/RentManagerTenantMapper.php
app/Modules/Integrations/RentManager/Mappers/RentManagerVendorMapper.php
app/Modules/Integrations/RentManager/Services/RentManagerOwnerService.php
app/Modules/Integrations/RentManager/Services/RentManagerService.php
app/Modules/Onboarding/Controllers/OwnerOnboardingController.php
app/Modules/Onboarding/Controllers/TenantOnboardingController.php
app/Modules/Onboarding/Controllers/VendorOnboardingController.php
app/Modules/Onboarding/Definitions/Contracts/OnboardingDefinition.php
app/Modules/Onboarding/Definitions/OwnerOnboardingDefinition.php
app/Modules/Onboarding/Definitions/TenantOnboardingDefinition.php
app/Modules/Onboarding/Definitions/VendorOnboardingDefinition.php
app/Modules/Onboarding/Services/OnboardingService.php
app/Modules/Onboarding/Services/OnboardingStatusResolver.php
app/Modules/Onboarding/Steps/Contracts/OnboardingStepsContract.php
app/Modules/Onboarding/Steps/OwnerOnboardingSteps.php
app/Modules/Onboarding/Steps/TenantOnboardingSteps.php
app/Modules/Onboarding/Steps/VendorOnboardingSteps.php
app/Modules/Onboarding/routes.php
storage/api-docs/api-docs.json
tests/Feature/Modules/Onboarding/OnboardingMeControllerTest.php
tests/Unit/Modules/Integrations/RentManager/Mappers/RentManagerOwnerMapperTest.php
tests/Unit/Modules/Onboarding/OnboardingServiceOwnerUpdateTest.php
tests/Unit/Modules/Onboarding/OnboardingStatusResolverTest.php
tests/Unit/Modules/Onboarding/Steps/OwnerOnboardingStepsTest.php
tests/Unit/Modules/Onboarding/Steps/TenantOnboardingStepsTest.php
tests/Unit/Modules/Onboarding/Steps/VendorOnboardingStepsTest.php
```

## Findings

### Blocker

**File:** `app/Modules/Integrations/RentManager/Mappers/RentManagerOwnerMapper.php`
- `fromOwnerData()` and `buildUpdatePayload()` no longer forward `is_active`, `property_ids`,
  `contacts`, or `primary_contact` to RRM, even though `HasOnboardingOwnerRules` still validates them,
  `OwnerData` still carries them, and the dev's own `RESTClient/onboarding.http` example still sends
  `property_ids: [901, 902]` / `contacts: []`. Silent no-op: 200 response, no effect in RRM.
- Snippet (`fromOwnerData`):
  ```php
  public static function fromOwnerData(OwnerData $data): array
  {
      $body = array_filter([
          'OwnerID' => $data->owner_id,
          'TaxID' => $data->tax_id,
          'Comment' => $data->comment,
      ], fn ($v) => $v !== null);
      $name = self::deriveName($data->is_company, $data->company_name, $data->first_name, $data->last_name);
      if ($name !== null) { $body['Name'] = $name; }
      $contact = array_filter([...], fn ($v) => $v !== null);
      if ($contact !== []) { $body['Contact'] = $contact; }
      if ($data->user_defined_values !== null) { $body['UserDefinedValues'] = $data->user_defined_values; }
      return $body;
  }
  ```
- Suggested fix: wire the fields back in (or route `property_ids` through the existing
  `linkOwnerProperties` flow), or remove them from validation/DTO if intentionally dropped.

### Should-fix

**File:** `app/Modules/Integrations/RentManager/Mappers/RentManagerOwnerMapper.php` (+ `OwnerOnboardingDefinition.php`)
- `toOwnerProfile()` reads `company_name` from `raw['CompanyName']`, a field the mapper's own docblock
  says doesn't exist on the RRM Owner root (only `Name, TaxID, Comment, Status, ...`), and the write path
  never sends `CompanyName` anymore (only derived `Name`). For a company owner (`is_company: true`),
  `company_name` will likely always be null, and `last_name` (from Contact) may also be null — meaning
  the OR-group `['last_name', 'company_name']` in `OwnerOnboardingDefinition::requiredFields()` will
  always report missing, reproducing the exact bug this PR claims to fix, but for company owners.
- No test exercises `toOwnerProfile`/`OwnerOnboardingDefinition` with `is_company: true`; the dev's own
  RESTClient ladder only tests an individual owner (Marcos Prosperi). Needs live verification against a
  company-type owner before merge.

### Nit

**File:** `app/Modules/Integrations/RentManager/Mappers/RentManagerOwnerMapper.php`
- `buildAddressPayloads`/`buildPhonePayloads` merge new rows onto existing RM collections by array
  index, not by `AddressID`/`PhoneNumberID`. Low risk for single-owner onboarding today, but fragile if
  an owner ever arrives with more than one pre-existing address/phone.

**File:** `RESTClient/onboarding.http`
- Typo in URL: `api/integrations-/rent-manager/custom/Roles/808/Privileges` (extra hyphen) breaks that
  example request.

## Verdict
needs changes before merge

## SPACE log
- Time spent reviewing: 40 min
- Model: Sonnet 5
- Findings: 1 blocker, 1 should-fix, 2 nits
- Verdict: needs changes before merge
