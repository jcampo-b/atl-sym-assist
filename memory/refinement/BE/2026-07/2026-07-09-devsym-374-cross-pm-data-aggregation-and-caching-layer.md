# DEVSYM-374 — [BE] Cross-PM data aggregation and caching layer
> Refined 2026-07-09, grounded in Claudio's repo diagnosis of `dev`. Source of truth is still the Linear ticket + AGENTS.md.

## Current ask (as read from Linear)
Aggregate cross-PM data for superusers and cache the slow tier (properties/units/tenants/owners) so RM (~1–2s/endpoint) isn't hit per request; fast tier (tickets/tasks) near-real-time; tag records by PM; expose counts. Body written against stale "PM API key per PM" and "no credential column" assumptions.

## Target
Repo(s): BE • Module: new SA aggregation/caching + Integrations/RentManager (wiring existing resolver) • Layer(s): SA domain • RM-backed: yes
Contract source: DEVSYM-392 empirical findings + repo diagnosis (existing cache precedent)

## Refined scope
Extend the existing SWR cache (Cache::flexible) keyed by rm_location_id instead of Auth::id(); slow-tier for properties/units/tenants/owners; per-record PM tag; counts/summaries; fast-tier reads with IsClosed,eq,false; on-demand ?refresh. NOT: fan-out (373), credential model (399), FE (375), session.

## Comment ledger
No comments on the ticket. Nothing to close out.

## Contradictions / open questions
- 373/374 seam — RESOLVED (2026-07-09, via 373 refinement):
  373's `CrossPmFanout::fanOut(Role, callable)` returns `array<int rm_location_id, array>` — one raw
  decoded-JSON array per reached connection (`RentManagerHttpClient::get()` returns `array`). 374 supplies
  the per-connection operation (the `callable`), caches per `rm_location_id`, then merges/tags/counts.
  373 never pre-merges — the per-Location split is exactly what makes 374's per-Location caching possible.
  → No longer blocked on contract definition; 374 still sequenced after 373 exists.
- Refresh path (→ Johnny, low-stakes): ticket grants "on demand OR on a schedule" (implementation freedom). Recommended lazy SWR + on-demand; scheduled warming deferred to a follow-up because it is the only path that adds an off-request RM caller + first scheduled job.

## Architecture decisions that apply
Reuse existing SA-domain cache (SWR Cache::flexible in PropertyPerformanceService/VacancyService; Cache::remember + version-counter in TaskService). Redis, no Cache::tags(). Key by rm_location_id, not Auth::id(). Integrations layer caches only the token. active = status='active' enum. Do not invent a cache mechanism (reuse over duplication). Scheduled/off-request path is a deferred boundary, not this ticket.

## Traps in play
Auth::id()→rm_location_id key change; no Cache::tags(); no webhooks (on-demand only); RM rate limits isolated per customer DB (DEVSYM-392) → a single fan-out pass (one request per Location) is fine, NO staggering/concurrency; Tenants+Units embed absent (two calls, join in SA); IsClosed,eq,false; deleted/stale records in RM active lists (filter in RM layer); rm_token_expires_at overload / 401-storm.

## Drift / flags
- Drift (ticket): "PM API keys" premise — disproven by 392, corrected in 399 + 373. Refined body applies the same correction.
- Drift (ticket + Claude memory): "no credential column" is FALSE. pm_connections.rm_credential_ref (encrypted text) stores {username,password}, used in PmConnectionContext.
- Drift (Claude memory): PartnerTokenContext is NOT a class — it's a credentialMode() branch in PmConnectionContext.
- Drift (Claude memory): prior note "374 = basic fan-out, no caching" contradicts 373 (owns fan-out) + 392 (cache mandatory) + the existing cache layer. Overridden.
- Architecture (conditional): only if scheduled warming is chosen (off-request RM caller + first job). Recommended path avoids it.

## Housekeeping
- pm_connections HAS rm_credential_ref (encrypted creds); 399 has UserSessionContext + PmConnectionContext (PartnerToken is a branch, not a 3rd class); 373 owns fan-out, 374 owns caching+counts.
- RULES.md gate added: for tickets depending on merged code, run repo-diagnosis BEFORE refining.

## Suggested estimate
5 pts recommended path (extend SWR to Location-keyed keys across 4 resources + counts + PM-tag, consume 373's fan-out, wire clientForConnection). 8 pts if scheduled warming is pulled in-scope — recommend a follow-up instead.
