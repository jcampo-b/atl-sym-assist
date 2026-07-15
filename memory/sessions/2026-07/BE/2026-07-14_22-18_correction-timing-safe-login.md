# Correction — timing-safe login (DEVSYM-415)

## What was wrong

`InternalUserAuthService::login()` short-circuited on a missing account:

```php
if ($user === null || $user->password === null || ! Hash::check($password, $user->password)) {
    abort(401, ...);
}
```

When the email did not exist (or the password was never set), the `||`
short-circuit returned BEFORE any `Hash::check`, so the failure was a fast-fail
(~3 ms) while a real credential check runs bcrypt (~270 ms). That timing
difference lets an attacker enumerate which internal-user emails exist.

## Correct approach

Always run one `Hash::check` on every path, comparing against a constant valid
bcrypt hash when there is no real one:

```php
private const DUMMY_HASH = '$2y$12$...'; // valid bcrypt, cost matches config

$knownHash = $user?->password;
$passwordMatches = Hash::check($password, $knownHash ?? self::DUMMY_HASH);

if ($user === null || $user->password === null || ! $passwordMatches) {
    abort(401, 'These credentials do not match our records.');
}
```

Measured medians (isolated `login()`, 5 iters): success 264.7 ms, wrong
password 270.6, unknown email 273.5, password-never-set 272.7, inactive 270.4 —
all within ~3%, dominated by the single bcrypt. No fast-fail, no reverse leak.

Note: `$user?->password ?? DUMMY` on one line trips Larastan's
`nullsafe.neverNull`; assign `$user?->password` to a variable first, then `??`.

## Rule going forward

Promoted to RULES.md → Code quality: any credential-checking login must run a
bcrypt comparison on every branch (dummy-hash fallback when no user/hash),
never short-circuit before the hash on a missing account.
