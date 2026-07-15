# Correction — fan-out over N RM targets must isolate each iteration

> DEVSYM-373 PR #139 review (jona872, CHANGES_REQUESTED), 2026-07-10.

## What was wrong
`CrossPmFanout::fanOut()` looped over active connections with no per-connection isolation:

```php
foreach ($this->policy->reachableConnections($role) as $connection) {
    $client = $this->resolver->clientForConnection($connection);
    $results[$connection->rm_location_id] = $operation($client);
}
```

`clientForConnection()` / `$operation($client)` can throw — e.g.
`RentManagerAuthenticationException` from `PmConnectionContext::decodeCredentials()` when a
connection's `rm_credential_ref` is null or malformed JSON. With no try/catch, a single broken
PM connection aborts the entire fan-out: none of the other active connections return a result.
This defeats the whole point of a superuser cross-PM view.

## Correct approach
Mirror the existing pattern in `OnboardingService::initTasks()` (looping over N independent
targets): wrap the loop body in `try/catch (\Throwable)`, emit a **sanitized** `Log::warning`
(`exceptionClass` = `$e::class` + `exceptionMessage` only — never the full Throwable, whose RM
response body may carry PII), skip the failed connection, and keep accumulating. Omitting the
failed connection's key is consistent with the `array<int rm_location_id, array>` contract.

## Rule going forward
Promoted to RULES.md → Layer discipline / RM Integration.
