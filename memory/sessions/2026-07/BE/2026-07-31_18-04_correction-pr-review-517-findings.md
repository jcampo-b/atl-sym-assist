# Correction — PR review DEVSYM-517 (PR #313), 3 findings

Recorded BEFORE applying any fix, per AGENTS.md's two-action protocol.
Source: `PR-reviews/2026-07/BE/2026-07-31_pr-review-devsym-517.md`.
Two findings valid (1 should-fix, 1 nit). One nit **rejected on evidence** — see finding 3.

## Finding 1 (should-fix) — silent empty planner on an unprovisioned PTR id

### What was wrong
`TaskService::pending()` returns an empty list when the acting connection's
`sa_ptr_rm_user_id` is null, with no log. That is byte-identical to "this superuser has no
pending tasks" — i.e. indistinguishable from the very bug DEVSYM-517 was reported as. And it
is not a rare edge: nothing populates the column yet, so every connection is in that state
at merge time.

### Correct approach
`Log::warning` in the internal-user branch only, carrying `pm_connection_id` and `rm_corpid`,
before falling into the shared `$rmUserId <= 0` return. Response payload unchanged.

### Rule going forward
**Nothing new to promote.** RULES.md → Code quality already carries this rule ("No silent
caps — a partial result returned because the 'impossible' case happened must be diagnosable,
not invisible", DEVSYM-374 PR #141 finding 2), and the reviewer cited it directly. This was a
failure to APPLY an existing rule, not a gap in it. Recorded here so the repeat is visible;
duplicating the rule in RULES.md would only dilute it.

## Finding 2 (nit) — a per-corpid value living on a per-connection table

### What was wrong
The migration docblock calls `sa_ptr_rm_user_id` "per-corpid", but `pm_connections` holds one
row per (corpid, location). `PmConnection::candidatesForCorpid()` / `firstForCorpid()` exist
precisely because a corpid can have several location rows. So the same PTR id has to be
written to EVERY row of that corpid. Miss one and a superuser acting for that location gets a
silently empty planner while the sibling location works — a genuinely nasty debug.

### Correct approach
Say so in the docblock, and make the follow-up discovery command write every
`candidatesForCorpid()` row rather than a single connection.

### Rule going forward
Promoted to RULES.md → "DEVSYM-399 foundation — as-built".

## Finding 3 (nit) — REJECTED: the `nullsafe.neverNull` citation is correct

### What the reviewer claimed
That `nullsafe.neverNull` only fires when the operand left of `?->` is non-nullable; that
since `ActingPmConnection::get(): ?PmConnection` is genuinely nullable, adding `?? 0` would
not trip it; and that the "Do NOT rewrite" comment is therefore unfounded institutional
memory. The reviewer explicitly noted they could not run PHPStan (no `vendor/` in the
checkout) and flagged rather than asserted it.

### What is actually true
Ran it. Wrote the counterfactual `(int) (app(ActingPmConnection::class)->get()?->sa_ptr_rm_user_id ?? 0)`
and ran `./vendor/bin/phpstan analyse app/Modules/Tasks/Services/TaskService.php` inside
Docker. It fails:

```
Using nullsafe property access "?->sa_ptr_rm_user_id" on left side of ?? is unnecessary.
Use -> instead.   nullsafe.neverNull
```

The receiver being nullable is irrelevant. The rule fires on the **syntactic position**: a
`?->` whose result is the left operand of `??` is reported as redundant, because `??` already
absorbs the null either way. Same trigger as the DEVSYM-415 case, so the RULES.md citation
was accurate — the reviewer misread that entry as being about narrowing.

### Rule going forward
Comment kept, wording sharpened to quote the actual error text and name the real mechanism
(`??` context, not receiver nullability) so the next reader doesn't re-derive the wrong
explanation. The mechanism is promoted to RULES.md alongside the existing DEVSYM-415 entry.

### Meta-lesson
A review finding that its own author marks as unverified ("I could not run PHPStan") is a
question, not a defect. Resolve it by running the tool, not by deferring to the reviewer —
and not by defending the original wording either. Here the claim was wrong but the
underlying worry (a directive comment hardening into institutional memory) was legitimate,
and it improved the comment.
