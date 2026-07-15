# Correction — PHPStan level 5 rejects Mockery in new test code

> DEVSYM-373, 2026-07-10. Caught by self-review PHPStan run before commit.

## What was wrong
`CrossPmFanoutTest` was first written with `Mockery::mock(...)` + `->twice()`/`->once()` and passed
the mock to a typed constructor. PHPStan level 5 raised 3 errors:
- `method.notFound` on `->twice()` / `->once()` (return type is `ExpectationInterface|HigherOrderMessage`).
- `argument.type`: `Mockery\MockInterface` given where the concrete class was expected.

Mockery mocks are not typed against the mocked class, so PHPStan can neither resolve the fluent
expectation methods nor accept the mock as the constructor argument. The captainhook pre-commit hook
runs `phpstan analyse` and would have blocked the commit (or forced a new baseline entry).

## Correct approach
Use PHPUnit's `$this->createMock(SomeClass::class)`. It returns `SomeClass&MockObject`, which:
- satisfies the constructor's type-hint (no `argument.type` error), and
- exposes the real class's method signatures so `->method()`, `->with()`, `->willReturn*()`,
  `->expects($this->exactly(n))` all type-check.

This is already the dominant convention in the repo (e.g. `TaskServiceUpdateTest`, `UserServiceLinkTest`
for its service dep). The only Mockery trace in `phpstan-baseline.neon` is legacy.

## Rule going forward
Promoted to RULES.md → Code quality.
