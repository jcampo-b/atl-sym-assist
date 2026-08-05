# PR Review — DEVSYM-452 (attention-needed pending regression)

## Review opened
- Timestamp: 2026-07-31
- Repo: BE
- Branch: `fix/devsym-452-attention-needed-pending-regression`
- Base branch: `origin/dev` (branch is 5 commits behind; deliberately NOT rebased per Johnny)
- Issue: DEVSYM-452
- Reviewed in git worktree `SymAssist-Backend-452-worktree` (main `SymAssist-Backend` dir owned by a live DEVSYM-517 session — hands-off)

## Dev's stated focus
Dashboard "Attention needed" block. Pending = not Completed AND not Scheduled AND due date not
passed (or none). Overdue = not Completed AND due date passed (includes Scheduled). Never both.

Root cause fixed: `TaskStatsService::DEFAULT_STATUS_IDS = [39]` was a raw RM status id hardcoded in
the SA layer; RM status ids are per-corpid. Fix routes through
`TaskStatusService::rmStatusIdsExcludingCompleted()` and consolidates the count + click-through
filter into `RentManagerIssueMapper::attentionClauses()`.

Requested review focus: layer discipline, whether count and click-through can drift, correctness of
the `DueDate,gen` operator choice, and whether tests exercise real paths.

## Files in scope
```
app/Modules/Dashboard/Services/AttentionNeededService.php
app/Modules/Integrations/RentManager/Mappers/RentManagerIssueMapper.php
app/Modules/Integrations/RentManager/Services/RentManagerIssueService.php
app/Modules/Integrations/RentManager/Services/RentManagerService.php
app/Modules/Tasks/DTOs/TaskListData.php
app/Modules/Tasks/Services/TaskService.php
app/Modules/Tasks/Services/TaskStatsService.php
tests/Feature/Modules/Dashboard/AttentionNeededPerCorpidTest.php
tests/Feature/Modules/Tasks/TaskStatsControllerTest.php
tests/Unit/Modules/Integrations/RentManager/Mappers/AttentionNeededPredicateTest.php
tests/Unit/Modules/Integrations/RentManager/Services/RentManagerIssueServiceTest.php
```
11 files, +663/-46.

## What checked out clean
- **`DueDate,gen` operator choice — verified, not assumed.** Queried live
  `ServiceManagerIssueFilterFields`: `DueDate` is `DateTime`, `IsNullable=True`, opcodes
  `bt eq ge gen gt gtn hv in le len lt ltn ne ni`. `gen` is supported. Also confirmed `StatusID`
  supports `in` (not `en`) and `IsClosed` only `eq`/`ne`.
- **Count/click-through convergence is structural, not incidental.** Both derive from
  `attentionClauses()`, and `AttentionNeededPredicateTest` asserts the two resolve to the same
  clause set. Mutual exclusivity is a property of `gen`/`lt` being exact complements, not of a
  caller remembering to subtract.
- **Tests exercise real paths.** `AttentionNeededPerCorpidTest` mocks only `RentManagerHttpClient`,
  so the real catalog service, name-matching, mapper and count service all run — unlike the
  pre-existing `AttentionNeededControllerTest`, which mocks `RentManagerService` (the service that
  was wrong) and passed throughout the bug's life.
- **Mutation-checked.** Swapping `gen`→`ge` fails 6 predicate cases, so the test is not vacuous.
- No raw RM PascalCase or status ids introduced outside `Integrations/RentManager/`. The original
  layer violation (`DEFAULT_STATUS_IDS = [39]`) is gone.

## Findings

### Blocker

**1. `app/Modules/Dashboard/Services/AttentionNeededService.php` — positive whitelist contradicts
452's "not Completed", and narrows Overdue vs production**

```php
        return Cache::flexible($key, self::CACHE_TTL, function () {
            $counts = $this->rentManager->countAttentionNeededTasks(
                $this->taskStatusService->rmStatusIdsExcludingCompleted(),
            );
```

Depends on (`app/Modules/Tasks/Services/TaskStatusService.php`):
```php
            $lifecycle = RentManagerIssueStatusMapper::lifecycleForName($status['Name'] ?? null);

            if ($lifecycle !== null && $lifecycle !== TaskStatus::Completed) {
                $ids[] = (int) ($status['ServiceManagerStatusID'] ?? 0);
            }
```

`rmStatusIdsExcludingCompleted()` is a POSITIVE whitelist of the 6 non-Completed SA lifecycle
statuses. 452 defines both buckets NEGATIVELY ("not Completed"). Tasks in RM statuses whose name
isn't an SA lifecycle name (RM's own `PENDING`, `Key disp`, any legacy status) now fall out of BOTH
counters even when open and past due.

Two consequences:
- Contradicts the ticket — an open legacy-status task past due IS Overdue per 452.
- REGRESSES Overdue vs `origin/dev`, where overdue is `IsClosed,eq,false;DueDate,lt,<today>` with no
  status constraint, so those tasks are counted today. The ticket said Overdue was not broken and
  must not change beyond what step 2 structurally requires.

Under-counting is the worst failure direction for a block whose purpose is surfacing work needing
attention.

Faithful form is available and tenant-correct: `StatusID` accepts `ne`/`ni` (verified), so use
`StatusID,ne,<thisConnectionsCompletedId>` via `rmStatusIdForLifecycle(TaskStatus::Completed)`.

Competing precedent: DEVSYM-443 deliberately chose a positive whitelist for the daily-planner pool
per a Lean product ruling — but that was scoped to the planner pool, not these counters. **Needs a
product call before merge, not a unilateral pick.**

### Should-fix

**2. `app/Modules/Tasks/Services/TaskStatsService.php` — extra RM round-trip**

```php
        $statusIds = collect($data->statusIds ?? $this->taskStatusService->rmStatusIdsExcludingCompleted())
            ->map(fn ($v) => (int) $v);

        $names = collect();
        if ($statusIds->isNotEmpty()) {
            $catalog = $this->rentManagerService->getServiceManagerStatuses();
            $names = collect($catalog['data'] ?? [])
```

Cold cache now fetches the status catalog twice in one request — once inside
`rmStatusIdsExcludingCompleted()` (cached 60 min), once directly for `$names` (uncached). Was one
call before this branch. Violates AGENTS.md "only re-fetch from RM when strictly necessary and
documented". Fix: reuse the catalog `TaskStatusService` already caches.

**3. `app/Modules/Integrations/RentManager/Mappers/RentManagerIssueMapper.php` — stale docblock now
self-contradictory**

```php
     * - `pending`   tickets in the RM "Pending" workflow status (see
     *               STATUS_PENDING). NOT date-based — RM has no way to
     *               filter `DueDate IS NULL` directly (empty value returns 422).
...
     * Buckets are NOT mutually exclusive: a ticket can be both `overdue`
     * and `pending` if it's past due AND in StatusID=STATUS_PENDING.
```

Three claims now false: `pending` IS date-based; the buckets ARE mutually exclusive; RM CAN filter
null due dates (`gen`). Sits ~40 lines above the new `attentionClauses()` docblock stating the
opposite, in a file this PR already modifies. Ticket scoped this out, so flagged not applied — but
this PR is what invalidated it.

**4. `app/Modules/Integrations/RentManager/Services/RentManagerIssueService.php` — filter-string
assembly in the Service**

```php
    public static function attentionFilters(string $bucket, array $notCompletedStatusIds): string
    {
        $clauses = RentManagerIssueMapper::attentionClauses($bucket);

        $statusClause = RentManagerIssueMapper::statusInClause($notCompletedStatusIds);
        if ($statusClause !== null) {
            $clauses[] = $statusClause;
        }

        return implode(';', $clauses);
    }
```

`implode(';', $clauses)` is Mapper work — `buildFilters()` next door owns it. Leaves two
filter-string assemblers in the RM layer, cutting against the PR's own "one place" goal. Move onto
`RentManagerIssueMapper`; `statusInClause()` could then go back to private.

### Nits

**5. `app/Modules/Tasks/Services/TaskService.php` — bucket detection disagrees with the mapper**

```php
        $isAttentionBucket = array_intersect(
            $data->schedule ?? [],
            [RentManagerIssueMapper::SCHEDULE_OVERDUE, RentManagerIssueMapper::SCHEDULE_PENDING],
        ) !== [];
```

Tests the whole array; `buildScheduleClauses()` honours only `$schedule[0]`. With
`schedule[]=completed&schedule[]=pending` the mapper emits `IsClosed,eq,true` while this injects
not-Completed ids — contradictory, returns nothing. Unreachable today (`'schedule' => [...,'max:1']`)
but `list()` is public. Read `$data->schedule[0] ?? null`.

**6. `app/Modules/Tasks/DTOs/TaskListData.php` — fragile named-arg spread**

```php
    public function withStatusIds(array $statusIds): self
    {
        return new self(...[...get_object_vars($this), 'statusIds' => $statusIds]);
    }
```

Silently depends on every property being a constructor parameter; a future non-constructor property
throws `Unknown named parameter` at runtime. Concise for a 27-param DTO, so a judgement call — at
minimum document the constraint.

**7. `tests/Unit/.../AttentionNeededPredicateTest.php` — asserts against a local RM model**

```php
            // `lt` never matches nulls.
            'DueDate,lt' => $task['due_date'] !== null && $task['due_date'] < $value,
            // `gen` == "greater-or-equal OR null".
            'DueDate,gen' => $task['due_date'] === null || $task['due_date'] >= $value,
```

Proves clauses match OUR model of RM, not RM. Mitigated (operator support verified live, semantics
documented, per-corpid test exercises the real stack, mutation-checked) — but a `gen` semantics
change in RM would not be caught. Worth naming in the docblock.

## Verdict
**needs changes before merge** — 1 blocker, 3 should-fix, 3 nits.

## Environment notes (not findings)
- Pre-existing on `origin/dev`, NOT introduced here: 10 test failures (9 = missing GD extension in
  the container image, 1 = `InternalUserPortfolioAccessTest`) and 1 PHPStan error
  (`VoiceAIService::SYSTEM_PROMPT_V3` unused, not in `phpstan-baseline.neon`).
- **`dev` cannot currently pass its own pre-commit hook** — that PHPStan error fails
  `captainhook`'s `phpstan analyse` action on a clean checkout. Worth a separate ticket: either
  baseline it or remove the constant.
- CaptainHook hooks shell into `docker compose exec app`, so they cannot run from a git worktree
  (wrong compose project). `--no-verify` does not skip `prepare-commit-msg`, which has the same
  wrapper; commits were made with an inline `-c core.hooksPath=<empty>` instead. Do NOT use
  `git config core.hooksPath` — `.git/config` is shared with the other worktree.
- Branch is 5 commits behind `origin/dev`; those 5 touch none of the files in scope.

## SPACE log
- Time spent reviewing: TBD
- Model: Opus
- Findings: 1 blocker, 3 should-fix, 3 nits
- Verdict: needs changes before merge
