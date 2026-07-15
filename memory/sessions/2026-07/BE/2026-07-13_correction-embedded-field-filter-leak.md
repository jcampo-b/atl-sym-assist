# Correction — embedded RM field filter leaking outside the RM layer

## What was wrong

While implementing DEVSYM-374's `PortfolioCacheService::fetchUnits()`, the
first draft embedded `Property` on the RM Units call and then filtered the
raw result client-side, reading `$unit['Property']['IsActive']` directly
inside `app/Modules/Superuser/Services/PortfolioCacheService.php` — a module
outside `app/Modules/Integrations/RentManager/`. This is the exact cardinal-
rule violation AGENTS.md already names ("If you see PascalCase field names
outside the RM layer → STOP"), just via an *embedded* field rather than a
top-level one, which made it easy to miss on a first pass.

## Correct approach

Moved the filter into a new `RentManagerUnitService::getActiveUnits()` method
(RM layer): it embeds `Property`, filters rows by `Property.IsActive`, and
returns already-filtered, still-PascalCase-but-RM-layer-contained data. The
SA-layer caller (`PortfolioCacheService`) only ever calls `getActiveUnits()`
and maps the result through `RentManagerUnitMapper::toUnit()` — it never reads
a raw RM field itself.

## Rule going forward

Any filtering logic that depends on a PascalCase RM field — whether top-level
or reached via an `embeds` relation — must live in a method inside
`app/Modules/Integrations/RentManager/` (Services or Mappers), never inline in
an SA-layer caller, even for a one-off/single-caller need. The existing
"confirm it's called from >1 place before adding a service method" rule does
NOT override this — the layer-discipline cardinal rule wins when the two
conflict.
