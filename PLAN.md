# Talkbox — Development Plan

Tracks progress through the 5-step planning protocol defined in CLAUDE.md, along with key decisions made at each stage.

---

## Protocol Progress

| Step | Description | Status |
|---|---|---|
| 1 | Clarify and bound requirements | Complete |
| 2 | Propose high-level architecture | Complete — approved 2026-04-06 |
| 3 | Break work into verifiable milestones | Not started |
| 4 | Write pseudo-code for milestone 1 | Not started |
| 5 | Implement incrementally with validation | Not started |

---

## Step 1 — Requirements (Complete)

**Use cases:** Sales and lead generation, customer support, appointment booking.

**Handler types:**
- Basic: Rule-based (keyword/pattern matching)
- Pro: LLM-powered (Claude Haiku)

**Volume:** ~250 conversations/day per tenant (~7,500/month). Plans structured for graceful scaling.

**Facebook Business Manager:** Always assume unverified at onboarding. If verified, it shortens the timeline.

**Languages:**
- Default: Spanish (all plans)
- Add-on: English language pack
- Add-on: French language pack
- Language packs are independent of handler type — available on both rule-based and LLM plans
- Opening message presented in all enabled languages simultaneously
- Bot defaults to Spanish; switches to detected language if supported by the tenant's plan
- Language detection uses `lingua-language-detector` with a minimum character threshold and confidence score

**Regions:**
- Mexico — Vultr Mexico / LFPDPPP
- USA — Fly.io (`dfw`) / CCPA + state laws
- Canada — DigitalOcean Toronto / PIPEDA + Quebec Law 25
- Each region is fully independent (separate hosting, DB, Redis, compliance docs)

---

## Step 2 — Architecture (Complete — approved 2026-04-06)

### Regional Deployment Model

Each region fully independent. Each tenant operates in exactly one region.

| Region | Host | Supabase | Redis | Secrets | Compliance |
|---|---|---|---|---|---|
| Mexico | Vultr Mexico | `us-east-1`* | Upstash | HashiCorp Vault (self-hosted) | LFPDPPP |
| USA | Fly.io (`dfw`) | `us-east-1` | Upstash | Fly.io Secrets | CCPA + state laws |
| Canada | DigitalOcean Toronto | `ca-central-1` | Upstash | DigitalOcean Secrets | PIPEDA + Quebec Law 25 |

*Closest available Supabase region at launch. Migration path in `docs/security/migration_path.md`.

Migrations applied independently per region as part of each region's deployment process. Procedure in `docs/operations/migration_procedure.md`.

**Upstash limits:** 10,000 commands/day and 256MB memory on the free tier. ~2 active tenants at 250 messages/day before paid tier required ($0.20/100K commands). Documented in tenant onboarding.

### Meta Account Model

Tenants own their own WABA, Meta app, and phone number. We assist with setup and optionally manage accounts on their behalf (`docs/operations/onboarding_meta.md`). Each tenant's `whatsapp_phone_number_id` and HMAC app secret stored per-tenant in Supabase.

**GET verification challenge:** Single global verify token used for Meta GET challenge — no phone number ID is present in the challenge so tenant resolution isn't possible. Per-tenant HMAC app secrets handle real security on all POST requests.

### Key Decisions

**Tenant model:** Multi-tenant single codebase. Tenant resolved from Meta phone number ID in webhook payload. All Redis keys namespaced by region and tenant ID (`{region}:tenant:{id}:user:{phone}:session`).

**Queue pattern:** `RPOPLPUSH` to a per-worker processing list for crash-safe message handling. Messages only removed from the processing list after successful send. Multiple worker instances safely consume from the same queue — horizontal scaling requires no architectural changes.

**Concurrency:** Per-user Redis lock before processing. Prevents race conditions when users send multiple messages quickly.

**Outbound send idempotency:** Before sending any reply, worker checks a Redis flag keyed to the incoming message ID (`{region}:tenant:{id}:sent:{message_id}`, 48-hour TTL). If the flag exists, send is skipped. Flag set immediately after successful send. Prevents duplicate replies on worker crash recovery.

**Outbound send queue (`app/meta/send_queue.py`):** All outbound Meta API calls go through a dedicated per-number send queue enforcing Meta's rate limits. A 429 from Meta re-queues with backoff — not treated as a failure, not written to dead letter. Only genuine delivery failures (5xx, timeout after retries) go to the failure path.

**Failure handling:**
- 3 retries with exponential backoff (2s, 8s, 32s)
- Failures tracked per tenant in a rolling 10-minute window (threshold: 5 = outage)
- Global counter tracks cross-tenant failures for platform-wide alerts
- Dead letter written to Supabase (encrypted, RLS, treated as PII)
- Canned error message sent to user on processing failures only

**Outage notifications:**
- SMS via Twilio to business owner (max one per 15 min, hard cap 3 per event)
- Plan holders: failure-type-specific message, we're resolving it, handle manually in the meantime
- Non-plan holders: same + nudge toward maintenance plan + affected time window (no PII)
- Internal team SMS for plan-holder outages
- Global threshold triggers internal-only alert
- Hourly platform-native cron re-pings internal team if outage open >2 hours

**Resolution:** `POST /internal/resolve-outage/{tenant_id}` (API key authenticated).
- Dead letter messages checked against Meta 24-hour window before re-processing
- Within window: re-queued via outbound send queue, chronologically, rate-limited
- Outside window: sent via pre-approved Meta template (`app/meta/templates.py`) or discarded with log entry
- Only most recent message per affected user re-processed (older messages are stale)
- "All clear" SMS sent only after re-processing completes without errors
- Non-plan holders receive affected time window summary for manual follow-up

**Language detection:** `lingua-language-detector`. Runs only when text ≥ minimum character threshold and confidence ≥ minimum score. Below either, current `user_language` preserved. Opening message assembled from all enabled locales at runtime with consent notice embedded.

**LLM context:** Token budget (not message count) caps conversation history. Hard 12-second timeout with `locales/{user_language}.json` fallback. Injection resistance via delimited system prompt structure. Per-user rate limit to prevent cost overruns.

**Compliance layer (`app/compliance/`):**
- `consent.py` — consent captured on first contact, embedded in multilingual opening message. Consent record stores timestamp, languages presented, compliance regime
- `retention.py` — per-regime retention limits enforced via nightly platform cron. Canada 72-hour breach notification tracked here
- `deletion.py` — keyword-triggered deletion requests purge all data from Supabase and Redis within legally required timeframe per regime

**Data persistence:**
- Rule-based tenants: completed interactions (leads, appointments, support) written to `interaction_log` in Supabase
- LLM tenants: full conversation history (token-capped sliding window), encrypted at rest
- Envelope encryption: per-tenant DEK encrypted by KEK in platform secrets (never environment variables)
- Annual key rotation standard; immediate rotation on suspected compromise or staff departure. Procedure in `docs/operations/encryption_key_rotation.md`
- Supabase connection pooling via PgBouncer explicitly configured
- Monthly cold storage export for compliance record retention beyond Supabase 7-day backup window

**Infrastructure:**
- Always-on: `min_machines_running = 1` on Fly.io, equivalent on DigitalOcean, external health-check ping on Vultr
- Scheduler runs as platform-native cron job — no third billable process
- Structured JSON logging shipped to log aggregator (Logtail or equivalent)
- Sandbox mode per tenant: processes normally but sends to test number
- Accepted risks at launch: Redis as single point of failure (Upstash redundancy sufficient), Supabase downtime (log and retry), Meta API version deprecation (not at launch)

--- 

## Step 3 — Milestones

*Not started. Will be defined after Step 2 is approved.*

---

## Step 4 — Pseudo-Code

*Not started.*

---

## Step 5 — Implementation Log

*Not started. Each milestone will be logged here with validation results.*

| Milestone | Status | Validated |
|---|---|---|
| — | — | — |

---

## Open Questions

*None currently.*

---

## Decisions Log

| Date | Decision | Rationale |
|---|---|---|
| 2026-04-05 | Independent regional deployments | Simplest path to data residency compliance per region |
| 2026-04-05 | Language packs independent of handler type | Avoids forcing LLM upgrade just to get English support |
| 2026-04-05 | `lingua-language-detector` over `langdetect` | Better accuracy on short text, actively maintained |
| 2026-04-05 | Token budget (not message count) for LLM history | Message count doesn't correlate to context window usage |
| 2026-04-05 | Manual outage resolution via internal endpoint | Avoids false "all clear" from automated detection |
| 2026-04-05 | Re-process only most recent message per user | Older outage messages are stale; re-sending them confuses users |
| 2026-04-05 | Platform-native cron for scheduler | Avoids third billable process on Railway/Fly.io |
| 2026-04-05 | Vultr Mexico / Fly.io USA / DigitalOcean Toronto | Cost, simplicity, and regional data residency alignment |
| 2026-04-05 | Tenant-owned WABA model | We don't own client Meta accounts; we help set them up and optionally help manage them |
| 2026-04-05 | Dedicated outbound send queue for Meta | 429s from Meta are throttling, not failures; need per-number rate awareness separate from dead letter |
| 2026-04-05 | Each tenant is single-region only | Small businesses operate in one country; no cross-region tenant coordination needed |
| 2026-04-05 | Always-on config required on all three platforms | Fly.io, Vultr, and DigitalOcean all risk cold starts that exceed Meta's webhook timeout |
| 2026-04-05 | Horizontal worker scaling via multiple instances | RPOPLPUSH + per-user locks already support this safely; needs to be stated explicitly |
| 2026-04-05 | Envelope encryption (DEK per tenant, KEK in platform secrets) | Rotation re-encrypts only DEKs, not all data; industry standard pattern |
| 2026-04-05 | Upstash free tier: 256MB memory + 10K commands/day | Both limits documented in onboarding; ~2 active tenants at 250 msg/day before paid tier |
| 2026-04-06 | Outbound send idempotency via Redis "reply sent" flag | Worker crash after LLM call but before processing list removal causes duplicate sends without this |
| 2026-04-06 | Single global verify token for Meta GET challenge | Per-tenant tokens can't be resolved on GET (no phone number ID in challenge); HMAC app secrets handle real security on POST |
| 2026-04-06 | HashiCorp Vault (self-hosted) for Vultr Mexico secrets | Vultr has no native secrets manager; Vault free tier is the correct solution for the Mexico deployment |
