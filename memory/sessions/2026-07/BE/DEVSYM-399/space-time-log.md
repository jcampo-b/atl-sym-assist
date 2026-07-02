DEVSYM-399 — [BE] Multi-tenant foundation — request-resolved RM credential context
Estimation: High priority, foundation ticket

Time breakdown:
  • Research (DEVSYM-392, RM API docs, architecture decision): 16 h
  • Context to Claude (armar el prompt, juntar links, correcciones del plan): 3 h
  • Claude execution + PR: ~60 min
  • Review modifications (IDE + 3 correcciones al plan): 1 h
  • Subtotal: ~21 h

Adoption: with intervention — 3 corrections applied during plan review
  (1) interface contract simplified: removed redundant refreshTokenAfterUnauthorized()
  (2) naming: Services/Context/ singular instead of Contracts/ + Services/Contexts/
  (3) TTL fix: CACHE_TTL_MINUTES = 10 replacing the incorrect 23h value

Model: Sonnet
