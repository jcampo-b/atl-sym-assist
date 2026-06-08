# Skill: Code Review

## When to read this
Before committing any implementation. Run this checklist against every file you touched.

## The rule of thumb
If you find yourself creating something new, ask first: does this already exist?
If you find yourself adding a parameter, ask: will this ever change, or is it a one-time fixed value?
If you find yourself writing a comment, ask: would the code be clear without it?

## Checklist — run before every commit

### No over-engineering
- [ ] Did you build only what the ticket asks for? No speculative abstractions.
- [ ] Did you add any interface, abstract class, or design pattern that isn't needed right now? Remove it.
- [ ] Are there any configuration values, constants, or injectable dependencies that will never vary? Inline them.

### No duplication
- [ ] Before creating a new service, mapper, or helper — did you check if an existing one already does this?
- [ ] Before adding a new method — does a method in the same class already do something equivalent?
- [ ] If you copied logic from another file: stop. Extract it instead, or use the original.

### Layer discipline
- [ ] Are there any PascalCase RM field names (`StatusID`, `DueDate`, etc.) outside of `app/Modules/Integrations/RentManager/`? If yes → move that logic to the RM layer.
- [ ] Is any SA module importing or depending on an RM class directly? It shouldn't.
- [ ] Is the controller doing anything beyond request validation, service delegation, and response formatting? Move that logic to a service.

### Validation layer discipline
- [ ] Is input validation (required fields, format, allowed values, conditional constraints) 
      in the FormRequest, not in the Service?
- [ ] Is there any guard in a Service or FormRequest that defends against a condition 
      that middleware already prevents (e.g. null user when auth:sanctum is active)? Remove it.
- [ ] Are you using FormRequest helpers (boolean(), integer(), string()) instead of 
      reading raw from $this->validated()? Use the helpers.

### RM round-trips
- [ ] Does the response re-fetch from RM ($this->get($id)) when the write method 
      already returns the updated data? Use the response directly unless there is 
      a documented reason for the re-fetch.

### State machine discipline  
- [ ] Is all transition validation — including exception paths like emergency bypass — 
      going through TaskStateMachine? The service should never inline transition guards.

### Code clarity
- [ ] Are there comments that just restate what the code does? Delete them.
- [ ] Are there comments that explain a non-obvious WHY? Keep them.
- [ ] Are there magic constants with semantic meaning (status IDs, timeout values that could change, category codes)? Name them.
- [ ] Are method and variable names self-explanatory without needing a comment?

### Laravel conventions (project-specific)
- [ ] Is business logic in a Service, not a Controller?
- [ ] Is RM field translation in a Mapper, not a Service?
- [ ] Are new providers registered in `bootstrap/providers.php` only?
- [ ] Is new middleware registered in `bootstrap/app.php` only?
- [ ] Are new scheduled commands in `routes/console.php` only?
- [ ] Did you use constructor promotion, `match`, enums, and `readonly` where applicable (PHP 8.3+)?

### Tests
- [ ] Does every new public method on a Service or StateMachine have a unit test?
- [ ] Does every new or modified endpoint have a feature test?
- [ ] Do all existing tests still pass? (`php artisan test`)

### API surface
- [ ] If you added or changed an endpoint — is there a runnable example in the relevant `.http` file under `RESTClient/`?
- [ ] If you changed OpenApi attributes — did you run `php artisan l5-swagger:generate`?
- [ ] If this PR changes an API contract — is the breaking change explicitly flagged in the PR description?

## Human review gate
After completing the self-review checklist above, stop.
Do not commit. Do not open a PR.

Notify the user that the implementation is ready for human review in their IDE.
Wait for explicit confirmation before proceeding to commit and PR.