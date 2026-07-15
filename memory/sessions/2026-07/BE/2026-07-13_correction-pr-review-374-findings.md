# Correction — DEVSYM-374 PR #141 review findings (3)

## Finding 1 (should-fix) — silent partial data in a cross-PM aggregation

### What was wrong
`PortfolioCacheService::compute()` derived `meta.counts` from `count(...)` of
the fanned-out rows. When `CrossPmFanout` skips a broken connection (its
documented "may return FEWER keys than reachableConnections()" contract), the
counts silently under-report and the response carries no signal that a PM was
dropped. For a cross-PM view, "PM is empty" and "PM #3 failed" look identical.

### Correct approach
Inject `SuperuserAccessPolicy`, compute the reachable set once, and expose
`meta.connections.{reachable, reached, skipped}`. A connection counts as
`reached` only if it succeeded for EVERY resource; any connection dropped for
at least one resource is `skipped`. `cachedResource()` now returns
`['rows' => ..., 'reached' => array_keys($perConnection)]` — the fan-out KEYS,
not the row count, so a reached-but-empty PM is still counted as reached.

### Rule going forward
See RULES.md → "Superuser stack ownership".

## Finding 2 (nit) — silent cap termination

### What was wrong
`fetchAllPages()` stops at `MAX_PAGES` defensively, but if the cap (not the
source `total`) ended the walk, it returned partial data with no log.

### Correct approach
Emit a `Log::warning` with resource + `rm_location_id` + `total` when the loop
exits via the cap while more pages are still owed.

### Rule going forward
See RULES.md → "Code quality" (no silent caps).

## Finding 3 (nit) — duplicate embed

### What was wrong
`RentManagerUnitService::getActiveUnits()` unconditionally appended `Property`
to `embeds`, yielding `embeds=Property,Property` if a caller already embedded
it. Diverges from the established `ALWAYS_EMBED` merge pattern
(`RentManagerIssueMapper` lines ~124-128) which guards with `str_contains`.

### Correct approach
Guard the append with `! str_contains($embeds, 'Property')`.

### Rule going forward
See RULES.md → "RM Integration" (embed-merge dedup).
