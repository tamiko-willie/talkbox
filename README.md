# Talkbox

WhatsApp chatbot platform for small businesses in Mexico, the USA, and Canada. Supports rule-based and LLM-powered bots with multilingual capability (Spanish, English, French) and independent regional deployments.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.12 + FastAPI |
| AI/LLM | Claude Haiku (Anthropic) |
| WhatsApp | Meta Cloud API |
| Database | Supabase (PostgreSQL + RLS) |
| Session/Queue | Upstash Redis |
| Notifications | Twilio SMS |
| Language Detection | lingua-language-detector |

## Regional Deployments

Each region is fully independent — separate hosting, database, Redis instance, and compliance documentation. No cross-region data access.

| Region | Host | Supabase Region | Compliance |
|---|---|---|---|
| Mexico | Vultr Mexico | `us-east-1`* | LFPDPPP |
| USA | Fly.io (`dfw`) | `us-east-1` | CCPA + state laws |
| Canada | DigitalOcean Toronto | `ca-central-1` | PIPEDA + Quebec Law 25 |

*Closest available Supabase region at launch. See `docs/security/migration_path.md` for upgrade path.

---

## Chatbot Plans

Two handler types, each available with optional language add-ons.

| | Rule-Based | LLM |
|---|---|---|
| Handler | Keyword/pattern matching | Claude Haiku |
| Memory | Session only (Redis) | Full history (Supabase) |
| Interactions logged | Yes (leads, appointments) | Yes (full conversation) |
| External integrations | No | Yes (CRM, calendar) |

**Language packs** (add-on per plan):
- Default: Spanish
- Add-on: English
- Add-on: French

---

## Project Structure

```
talkbox/
├── app/
│   ├── main.py                  # FastAPI entry point
│   ├── config.py                # Settings via environment variables
│   ├── tenants.py               # Tenant resolution from phone number ID
│   ├── auth.py                  # Internal API key authentication
│   ├── locks.py                 # Per-user Redis concurrency locks
│   ├── ratelimit.py             # Per-user LLM rate limiting
│   ├── scheduler.py             # Outage check script (run via platform cron)
│   │
│   ├── webhook/
│   │   ├── router.py            # POST /webhook, GET /webhook
│   │   ├── internal_router.py   # POST /internal/resolve-outage
│   │   ├── health_router.py     # GET /health
│   │   └── signature.py         # X-Hub-Signature-256 verification
│   │
│   ├── queue/
│   │   ├── producer.py          # RPOPLPUSH onto Redis processing list
│   │   ├── consumer.py          # Async worker main loop
│   │   ├── dead_letter.py       # Failed message storage and retry logic
│   │   └── reprocessor.py       # Re-queues dead letter on outage resolution
│   │
│   ├── handlers/
│   │   ├── rule_based.py        # Keyword/pattern matching handler
│   │   └── llm.py               # Claude Haiku handler with token window
│   │
│   ├── meta/
│   │   ├── client.py            # Meta Cloud API client (send, typing indicator)
│   │   └── templates.py         # Pre-approved Meta message templates
│   │
│   ├── db/
│   │   ├── supabase.py          # Supabase client with PgBouncer pooling
│   │   └── models.py            # Table schemas and query helpers
│   │
│   ├── session/
│   │   └── redis.py             # Per-user session state (user_language, flow)
│   │
│   ├── security/
│   │   └── encryption.py        # Customer data encryption at rest
│   │
│   ├── compliance/
│   │   ├── consent.py           # Consent capture and storage (LFPDPPP/CCPA/PIPEDA)
│   │   ├── retention.py         # Data retention enforcement per regime
│   │   └── deletion.py          # User deletion request handling
│   │
│   ├── notifications/
│   │   ├── sms.py               # Twilio SMS client
│   │   └── alerting.py          # Outage notification logic (owner + internal team)
│   │
│   ├── middleware/
│   │   └── rate_limit.py        # IP-based rate limiting on webhook endpoint
│   │
│   ├── utils/
│   │   └── phone.py             # E.164 phone number normalisation
│   │
│   └── locales/
│       ├── es.json              # Spanish responses (default)
│       ├── en.json              # English responses
│       └── fr.json              # French responses
│
├── worker/
│   └── run_worker.py            # Worker process entry point
│
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_conversations.sql
│   ├── 003_create_dead_letter_queue.sql
│   ├── 004_rls_policies.sql
│   ├── 005_create_outage_log.sql
│   ├── 006_create_consent.sql
│   ├── 007_create_interaction_log.sql
│   └── 008_add_tenant_region.sql
│
├── tests/
│   ├── test_webhook.py
│   ├── test_signature.py
│   ├── test_queue.py
│   ├── test_rule_based.py
│   └── test_meta_client.py
│
├── deploy/
│   ├── mexico/
│   │   ├── vultr.toml           # Web + worker processes
│   │   └── vultr_cron.toml      # Scheduler cron job
│   ├── usa/
│   │   ├── fly.toml             # Web + worker processes
│   │   └── fly_cron.toml        # Scheduler cron job
│   └── canada/
│       ├── do_app_spec.yaml     # Web + worker processes
│       └── do_cron.yaml         # Scheduler cron job
│
├── docs/
│   └── security/
│       ├── lfpdppp_compliance.md
│       ├── ccpa_compliance.md
│       ├── pipeda_compliance.md
│       ├── backup_retention_policy.md
│       ├── migration_path.md
│       └── security_practices.md
│
├── .env.example
├── .gitignore
├── Dockerfile
├── Procfile
├── requirements.txt
└── CLAUDE.md
```

---

## Environment Variables

See `.env.example` for the full list. Required variables per deployment:

```
META_VERIFY_TOKEN=
META_APP_SECRET=
SUPABASE_URL=
SUPABASE_KEY=
REDIS_URL=
ANTHROPIC_API_KEY=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_FROM_NUMBER=
INTERNAL_API_KEY=
INTERNAL_TEAM_PHONE=
ENCRYPTION_KEY=
REGION=                # mexico | usa | canada
```

---

## Running Locally

```bash
# Install dependencies
pip install -r requirements.txt

# Copy and fill in environment variables
cp .env.example .env

# Start the web server
uvicorn app.main:app --reload

# Start the worker (separate terminal)
python worker/run_worker.py
```

---

## Compliance

Each regional deployment operates under a separate compliance regime. See `docs/security/` for documentation on LFPDPPP (Mexico), CCPA (USA), and PIPEDA + Quebec Law 25 (Canada), data retention policies, and the cloud migration path for data residency requirements.

---

## Reliability

- Webhook returns `202 Accepted` immediately — no LLM or DB calls on the request thread.
- Messages processed asynchronously via Redis queue with crash recovery (`RPOPLPUSH`).
- Failed messages retried 3 times (2s / 8s / 32s backoff) before dead letter.
- Outage threshold: 5 failures within 10 minutes triggers owner SMS notification.
- Business owners notified via SMS on failure; internal team notified for plan holders.
- Dead letter messages re-processed on manual resolution, with Meta 24-hour window check.
