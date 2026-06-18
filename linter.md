## Description

Adds code quality tooling to the backend (SYM-150 / SYM-151 / SYM-152).
Three tools that run automatically before every commit to keep the codebase consistent and catch bugs early. 
The bulk of the diff is the initial one-time reformat of the existing codebase by Pint. 
All devs **must** run **`composer install`** after pulling. This installs the pre-commit git hooks locally (via CaptainHook). Until you do this, nothing is enforced on your machine.
 

## What we added

### Pint — code style formatter
Laravel's official formatter. Enforces a consistent code style (imports, spacing, etc.) across the whole repo. Config in `pint.json`.

**How to use:**
- It runs automatically on every commit (check-only). If a commit is blocked,
  run the fixer and re-commit:
  ```bash
  ./vendor/bin/pint     # auto-fixes formatting
  git add .
  git commit
- `./vendor/bin/pint --test` → check only, fixes nothing (what the hook runs).

Recommended: enable format-on-save in your editor so commits never get blocked (see "Editor setup" below).

### PHPStan / Larastan — static analysis

Reads your code without running it and catches bugs: undefined methods, wrong types, possible nulls, etc. 
Running at level 5. Config in phpstan.neon.
Pre-existing issues are frozen in phpstan-baseline.neon, so the hook only blocks you on new issues you introduce — not legacy debt.

**How to use:**
- .`/vendor/bin/phpstan analyse --memory-limit=512M`
- If you fix legacy issues and want to update the baseline:
- `./vendor/bin/phpstan analyse --generate-baseline`

### CaptainHook — git pre-commit hooks

Wires the above into git. On every git commit it runs Pint (--test) + PHPStan automatically. If either fails, the commit is aborted. Config in `captainhook.json`. Installed automatically by composer install. (work on background, no action needed)


## Editor setup (recommended)

```
{
    "[php]": {
        "editor.defaultFormatter": "open-southeners.laravel-pint",
        "editor.formatOnSave": true
    },
    "laravel-pint.configPath": "pint.json",
    "laravel-pint.enable": true
}

```

## How to test

```
./vendor/bin/pint --test
./vendor/bin/phpstan analyse --memory-limit=512M

```
Both should pass with no errors.



**Base repos:**
[Pint](https://github.com/laravel/pint)
[PHPStan](https://github.com/phpstan/phpstan)
[CapitanHook](https://github.com/captainhookphp/captainhook)

---
⚠️  **Do not merge before May 30** — first major deliverable in progress, this will block commits with formatting/type errors.

  Adds code quality tooling to the Backend repo (3 commits):
  - `pint.json` — Laravel code style formatter with initial codebase formatting
  - `phpstan.neon` — Static analysis at level 5 (larastan), with baseline for pre-existing issues
  - `captainhook` — Pre-commit hooks that run Pint and PHPStan before every commit

  ## ⚠️  After merging

  All devs must run `composer install` to activate the pre-commit hooks locally.
  From that point, commits with formatting or type errors will be blocked.