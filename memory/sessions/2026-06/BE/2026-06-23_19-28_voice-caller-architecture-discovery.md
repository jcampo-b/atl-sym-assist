# Voice Caller — Architecture Discovery & RAG Deep Dive

- **Date:** 2026-06-23
- **Branch:** `feature/twilio-v2` @ `b0da557` (NOT merged to `dev`)
- **Scope:** read-only discovery. No application files modified.
- **Repo:** `SymAssist-Backend/`

---

## 0. Executive summary — what the Voice Caller actually is

The Voice Caller is **not** a document-knowledge RAG system with ElevenLabs streaming.
The brief's framing was based on assumptions that the code does not support. Ground truth:

- **Turn-based phone IVR over Twilio.** Inbound call → Twilio webhook → Laravel returns
  TwiML (`<Gather input="speech">` + `<Say>`). TTS is Twilio's own `<Say>` (voice `alice`).
  **No ElevenLabs. No WebSocket. No media streaming.** Confirmed absent everywhere.
- **OpenAI LLM drives the conversation** (`gpt-4o` for chat, `gpt-4o-mini` for routing/discovery,
  `text-embedding-3-small` for embeddings).
- **The "RAG" is intent→tool-pattern matching, not knowledge retrieval.** Embeddings index
  short natural-language *question patterns* and *RentManager API endpoint descriptions*, so the
  system can map a caller's spoken intent to a safe read-only RRM API call. There is **no
  document chunking, no PDF ingest, no transcript storage, no knowledge base.**
- **Two parallel call paths** that share only `RentManagerService`:
  - `VoiceBalance` (legacy) — DTMF key-press, balance-only, single-turn.
  - `VoiceAI` (active) — speech, multi-turn, LLM-powered. Has a V1 path (discrete tools) and a
    V2 path (RAG discovery pipeline) gated by the `VOICEAI_DYNAMIC_DISCOVERY` flag (default OFF).

### Corrections to the brief's assumptions (verify these before repeating them)
| Brief assumed | Reality on `feature/twilio-v2` |
|---|---|
| ElevenLabs + WebSocket streaming | None. Twilio `<Say>`/`<Gather>`, turn-based. |
| Document-knowledge RAG (chunking, top-k over docs) | Embeddings over intent patterns + API-method descriptions only. |
| import / export / manual-training artisan commands | Only two: `voice-ai:embed-patterns`, `voice-ai:seed-methods`. |
| "Jona left a manual in docs/" | No voice/Twilio markdown doc exists. Knowledge lives in `RESTClient/twilio*.http` + code. |
| PostgreSQL replaced Redis for RM tokens | **Redis is still used** (cache/session/queue). RM token cached in Redis 23h, single global key. |
| Self-learning improves from transcripts | Pattern confidence is nudged ±0.05/0.10 per call success/failure. Transcripts are not stored. |

---

## 1. Module / file map (introduced by VC)

```
app/Modules/VoiceAI/
  Services/        VoiceAIService, DiscoveryPipeline, IdentityResolver, IntentRouter,
                   LlmClientInterface, OpenAiLlmClient, ClaudeLlmClient, FakeLlmClient
  Repositories/    EmbeddingClient, ToolPatternRepository, RrmMethodRepository
  Tools/           GetBalanceTool, GetTenantInfoTool
  Commands/        EmbedPatternsCommand, SeedRrmMethodsCommand
  Jobs/            UpdateToolPatternJob, HumanReviewJob
  Controllers/     VoiceAIController, VoiceAIDevController
  routes.php, VoiceAIServiceProvider
app/Modules/VoiceBalance/
  Controllers/     VoiceCallController, VoiceDevController
  Services/        VoiceBalanceService
  routes.php, VoiceBalanceServiceProvider
app/Modules/Integrations/Twilio/Services/  TwilioClient, TwilioVoiceService
database/migrations/  000001 rrm_api_methods, 000002 voice_ai_tool_patterns,
                      000003 drop_ivfflat_indexes, 000004 add_entity_type (dated 05_26)
config/  ai.php, openai.php, twilio.php, voiceai.php
infra/   nixpacks.toml, railway.toml, .php-version ; config/horizon.php + HorizonServiceProvider REMOVED
RESTClient/  twilio.http, twilio-ai.http, twilio2.http
```

---

## 2. Branch evolution — "why it is the way it is"

(Local `v1`/`v1.5` refs are gone; narrative recovered from `git log dev..feature/twilio-v2`
and the `buckup/twilio-v1_14-05` snapshot.)

### v1 — get a real call working
DTMF `<Gather numDigits=1>` IVR, balance-only, **tenant-only** (owner lookup deliberately
deleted at `48f4cb4`). Phone stripped to 10 digits to match RRM `StrippedPhoneNumber`.
`<Pause length=1>` before `<Say>` so callers don't lose the first word. `voice_balance`
cache for concurrent calls. `trustProxies` added for ngrok HTTPS (signature validation).

### v1.5 — the Railway deploy crisis
Nothing deployed first try. Fix sequence: add `.php-version`, add `nixpacks.toml`, **remove
Horizon** (`2667399` "RW deploy fix: remove horizon" — its long-lived daemon is incompatible
with Railway's single `startCommand`), bump PHP `^8.2 → ^8.4`, add `railway.toml`
(`php artisan migrate --force && /start-container.sh`). Added `SwaggerAuth` middleware.
**Reversal:** Horizon was a planned dependency in v1, cut under deploy pressure. Redis itself
stayed.

### v2 — the VoiceAI module (bulk of the work, ~3.6k net lines)
- **2a Skeleton + multi-provider LLM:** `LlmClientInterface` with OpenAI/Claude/Fake impls;
  `config/ai.php` `LLM_PROVIDER` selects at runtime. OpenAI primary, Claude secondary.
- **2b V1 AI path:** `GetBalanceTool`/`GetTenantInfoTool` as OpenAI function-calling tools;
  conversation state in Cache keyed by `CallSid`; speech-turn loop.
- **2c V2 AI path (RAG):** `rrm_api_methods` + `voice_ai_tool_patterns` tables with
  `vector(1536)`; `EmbeddingClient` (text-embedding-3-small); `DiscoveryPipeline` (cosine
  search → pattern reuse, else dynamic RRM discovery); `IntentRouter`; `IdentityResolver`;
  async learning jobs; seed/embed commands.
  - **Reversal:** ivfflat indexes created in 000002 then **immediately dropped** in 000003 —
    ivfflat builds centroids at index-create time; on empty tables that yields 0 centroids and
    every search returns 0 rows. Decision: run seq-scans now, re-add ivfflat at 5,000+ rows
    with `lists = sqrt(n)`.
- **2d Multi-entity:** `entity_type` column (tenant/owner/vendor) added to both tables;
  existing patterns backfilled as `tenant`; owner patterns whitelisted. **Partial reversal of
  the v1 owner-removal** — owners return, but only in the VoiceAI path; legacy VoiceBalance
  still tenant-only. `expires_at` TTL (90 days) for unused patterns.

### Dependency delta
`laravel/horizon ^5.43` removed · `openai-php/client ^0.19.2` added · `twilio/sdk ^8.11`
unchanged · PHP min `^8.2 → ^8.4`. **No pgvector PHP package** — all vector ops are raw SQL.

---

## 3. RAG architecture deep dive (primary learning objective)

### Storage
Two pgvector tables (`CREATE EXTENSION IF NOT EXISTS vector`), both with an
`embedding vector(1536)` column added via raw `ALTER TABLE`:

- **`rrm_api_methods`** — registry of safe read-only RRM GET endpoints the LLM may discover
  against. Key cols: `endpoint`, `base_path`, `http_method`, `description` (LLM-generated),
  `params_schema`/`response_schema` (jsonb), `requires_entity_id`, `is_read_only` (def true),
  `is_whitelisted` (def **false** — manual gate), `risk_level`, `embedding`.
- **`voice_ai_tool_patterns`** — learned/seeded intent patterns. Key cols: `question_pattern`
  (the embedded text), `rrm_method_id` FK, `endpoint`, `params_template` (with `{entity_id}`),
  `field_map`, `response_shape`, `confidence` (start 0.5), `success_count`, `failure_count`,
  `is_human_reviewed`, `pii_safe`, `last_used_at`, `expires_at`, `embedding`.
- `entity_type` (000004) scopes both tables tenant/owner/vendor.

**Distance metric:** cosine via pgvector `<=>`. Similarity returned as `1 - (embedding <=> ?::vector)`.

### Embedding generation
- Model **`text-embedding-3-small`**, explicit `dimensions: 1536` (`config/voiceai.php`).
- Call site: `EmbeddingClient::embed()` using **Laravel `Http` facade** (NOT openai-php, NOT
  Guzzle directly) → `POST https://api.openai.com/v1/embeddings`, 10s timeout.
- Embedded text: patterns → the raw `question_pattern` bag-of-synonyms string; methods → an
  LLM-generated (`gpt-4o-mini`) one-sentence endpoint description.
- **No cache/dedup** beyond `WHERE embedding IS NULL` (skips already-embedded rows).
- Vector serialized as `'[' . implode(',', $v) . ']'` cast to `::vector` in SQL.

### Retrieval (query time)
`DiscoveryPipeline::execute()` embeds the caller intent, then
`ToolPatternRepository::searchSimilar()`:
```sql
SELECT ..., 1 - (embedding <=> ?::vector) AS similarity
FROM voice_ai_tool_patterns
WHERE (expires_at IS NULL OR expires_at > NOW()) AND embedding IS NOT NULL
  AND (entity_type = ? OR entity_type IS NULL)
ORDER BY embedding <=> ?::vector LIMIT ?
```
- **top-k:** 3 for patterns, 5 for RRM methods.
- **Thresholds** (`config/voiceai.php`): `high_confidence 0.80`, `medium_confidence 0.50`
  (min to reuse a pattern), `human_review 0.20` (pattern confidence, not similarity).
- Flow: top-1 similarity ≥ 0.50 → reuse pattern; else → dynamic discovery against
  `rrm_api_methods` (top-5 candidates → `gpt-4o-mini` proposes endpoint → **whitelist + must-be-in-candidates
  guard prevents hallucinated endpoints** → execute → `savePattern()` for future reuse).
- No reranking, no hybrid/BM25.

### Ingestion (artisan)
- **`voice-ai:embed-patterns`** — seeds ~17 hardcoded tenant/owner patterns (confidence 1.0,
  human-reviewed, permanent), then embeds every row with NULL embedding. No chunking.
- **`voice-ai:seed-methods [--dry-run]`** — iterates 14 curated RRM resources, pulls real API
  method metadata, filters to GET-only, has `gpt-4o-mini` write a one-sentence description,
  embeds it, upserts by `endpoint`. **All rows seed with `is_whitelisted=false`** — a human
  must flip the flag before an endpoint participates in discovery.

### Self-learning
- `UpdateToolPatternJob` (async, dispatched per execution): success → `confidence += 0.05`
  (cap 1.0), `success_count++`, TTL reset +90d; failure → `confidence -= 0.10` (floor 0),
  `failure_count++`. If confidence < 0.20 → dispatch `HumanReviewJob`.
- `HumanReviewJob` is a **stub** — only `Log::warning`. No email/Slack/UI, no auto-delete.
- No transcript storage, no quality scoring beyond the confidence counter.

---

## 4. PHP implementation notes (how RAG-in-PHP was done)

| Concern | Implementation |
|---|---|
| OpenAI embeddings | Laravel `Http` facade → REST `/v1/embeddings`. |
| OpenAI chat (main) | `openai-php/client` SDK in `OpenAiLlmClient` (model/max_tokens from `config/ai.php`, `tool_choice: auto`). |
| OpenAI chat (routing/discovery) | Raw `Http` facade to `gpt-4o-mini`, `response_format: json_object`; independent of `LLM_PROVIDER`. |
| Claude alt | `ClaudeLlmClient` via `Http` to Anthropic `/v1/messages`, converts OpenAI↔Anthropic message+tool shapes. |
| pgvector | **Raw SQL only** — `DB::statement()`/`DB::select()` with `?::vector`. No Eloquent model/cast, no pgvector package. |
| Prompt assembly | `VoiceAIService::buildMessagesV1/V2()` = `[system] + history[]`. V2 system prompt injects resolved `entity_type/id/name` and exposes a single virtual `queryRRM` tool. |
| Provider binding | `VoiceAIServiceProvider` `match(config('ai.provider'))` → claude / openai (Fake if no key) / Fake. Bound transient. |

**Gotcha:** V1 tools (`GetBalanceTool`/`GetTenantInfoTool`) are **singletons** with mutable
`setPhone()` state set per turn — fine single-threaded, risky if reused concurrently.

---

## 5. Infra & integration map

- **Twilio:** webhook `POST /api/twilio/ai/voice` → `VoiceAIController` (validates
  `X-Twilio-Signature` via `TwilioClient` + SDK `RequestValidator`) → `VoiceAIService::handle`.
  Legacy `POST /api/twilio/voice` → `VoiceCallController`. Responses are TwiML `<Gather>`/`<Say>`.
  `TwilioClient` also does `updateVoiceWebhook()` and outbound `createVoiceCall()` (dev).
- **Railway:** `nixpacks.toml` pins PHP 8.4 + extensions (incl. `redis`, `pdo_pgsql`);
  `railway.toml` runs migrations on every deploy. pgvector expected on Railway's managed PG.
- **RM token cache:** `RentManagerAuthService` — `Cache::put('rent_manager:api_token', …, 23h)`.
  Single global key, Redis-backed (`bootstrap/redis-env.php` falls back to file cache only for
  non-Docker local dev when Redis is unreachable). **Redis was NOT replaced by Postgres.**
- **Horizon removed**, but Redis remains the cache/session/queue backend; jobs run on the
  default queue worker, not Horizon's supervisor.
- **New env:** `TWILIO_ACCOUNT_SID/AUTH_TOKEN/PHONE_NUMBER/SUPPORT_PHONE` (+ unused
  `TWILIO_API_KEY` leftover), `OPENAI_API_KEY/MODEL(gpt-4o)/MAX_TOKENS(150)`.

---

## 6. V3 multi-tenant plan assessment (vs current code)

V3 goal: a second PM operates independently without code changes (Twilio subaccount per PM,
encrypted creds, invite onboarding). 14 V3.0 tasks (~56h).

### Implemented vs greenfield
| # | Task | Status in v2 |
|---|---|---|
| 1 | `voice_clients` table | **Greenfield.** No table/code. |
| 2 | `vc_invites` table | **Greenfield.** |
| 3 | `VoiceClientContext` | **Greenfield.** |
| 4 | `ResolveVoiceClientMiddleware` | **Greenfield.** No per-call PM resolution. |
| 5 | `RentManagerHttpClient` refactor (per-PM creds) | **Greenfield + blocker.** Today it's a **singleton** reading global `config('rent_manager.*')`. Not injectable per-PM. |
| 6 | `RentManagerAuthService` per-PM token | **Greenfield.** Single global cache key `rent_manager:api_token`. Must become per-PM keyed. |
| 7 | `VoiceAIController` ties call→PM | **Partial.** Controller + identity resolution exist, but no PM dimension. |
| 8 | `VoiceAIWebSocketHandler` | **Greenfield AND mis-specified.** v2 has no WebSocket; it's turn-based `<Gather>`. This task assumes an architecture that does not exist — confirm intent before estimating. |
| 9 | `VoiceAIService` + `TwilioClient` per-PM creds | **Greenfield.** Both read global config today. |
| 10 | `TwilioProvisioningService` | **Greenfield.** No provisioning/subaccount code anywhere. |
| 11–13 | Admin invite UI + onboarding form | **Greenfield** (backend; FE separate). |
| 14 | Migration of existing config to multi-PM | **Greenfield.** Requires backfilling current single-tenant config into `voice_clients`. |

### Risks (Jona's + observed)
- **Jona flagged:** #5 touches the core (full regression needed); #10 makes real Twilio calls
  (test mode first); #14 zero-downtime in prod.
- **Additional, from the code:**
  - **#5 is the linchpin.** Every RRM call funnels through the singleton `RentManagerHttpClient`
    + global-keyed `RentManagerAuthService`. Making creds/token per-PM is a cross-cutting change
    touching auth caching, the HTTP client, and request-scoped context (#3/#4) simultaneously.
  - **#8 contradicts the current architecture** — no WebSocket exists; the AI session is
    stateless-per-turn over Cache keyed by `CallSid`. Multi-tenant state should hang off that
    Cache/context, not a WS handler. The 4h estimate is for code that may not be the right shape.
  - **pgvector tables are not PM-scoped.** `voice_ai_tool_patterns`/`rrm_api_methods` have no
    `voice_client_id`. If patterns must be isolated per PM, that's an extra column + migration
    not listed in V3.0 (today only `entity_type` exists).
  - **`is_whitelisted` is manual.** Per-PM dynamic discovery means a manual whitelist step per PM
    unless automated — friction for "no touching code" onboarding.
  - **Redis dependency.** Per-PM token caching rides on Redis; Railway Redis sizing/availability
    becomes a multi-tenant scaling concern.

---

## 7. Open questions / follow-ups
- Is `VOICEAI_DYNAMIC_DISCOVERY` (V2 RAG path) intended to ship ON, or stay behind the flag?
- Should V3 pattern tables be PM-scoped (`voice_client_id`) or shared across PMs?
- V3 task #8 (`VoiceAIWebSocketHandler`): is a streaming rewrite actually planned, or is the
  task a leftover from an earlier streaming design? Current path is turn-based.
- `HumanReviewJob` is a stub — confirm whether a real review channel is needed before relying on
  the self-learning loop in production.
