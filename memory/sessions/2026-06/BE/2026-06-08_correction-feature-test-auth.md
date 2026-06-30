# Correction — Feature test auth with BypassAuthForDev middleware

## What was wrong
Feature tests using actingAs($user, 'sanctum') were returning 500 on routes
that access $request->user()->email or $request->user()->id.

Root cause: BypassAuthForDev middleware runs in Docker (APP_ENV=local,
DEV_AUTH_BYPASS=true). When no bearer token is present in the request, the
middleware overrides actingAs() by calling Auth::guard('sanctum')->setUser()
with a DB-looked-up user (rm_user_id from config). That user may have null
email/id depending on DB seed state, corrupting the authenticated user even
when actingAs() was correctly called.

## What the correct approach is

Option A (used in DEVSYM-152): pass a fake bearer token in test headers.
The middleware skips when a bearer token is present (! $request->bearerToken()).
The value is irrelevant — Sanctum with actingAs() never validates it.

  private function authHeaders(): array
  {
      return ['Authorization' => 'Bearer test-token'];
  }

  $this->actingAs($user, 'sanctum')
      ->patchJson('/api/tasks/42/status', $payload, $this->authHeaders())

Option B (cleaner, preferred going forward):

  $this->withoutMiddleware(\App\Http\Middleware\BypassAuthForDev::class);

Option B is explicit about intent and does not depend on middleware internals.
Use it in any new feature test that accesses request->user() properties.

## Always use create(), not make()
User::factory()->make() does not persist the user. Guards that do DB lookups
will not find it. Always use User::factory()->create() in feature tests.

## Rule to apply going forward
Any feature test that accesses authenticated user properties (email, id, etc.)
must either use withoutMiddleware(BypassAuthForDev::class) or pass a bearer
token in headers. Tests that only check HTTP status codes and do not access
user properties are unaffected.
