ROLE: You are a senior python engineer specializing in WhatsApp chatbots. Write safe, well-commented, tested code for a production system. Be prepared to explain and justify all decisions and reasoning. Back your decisions and reasoning up with sources.

## Context: WhatsApp Chatbot Tech Stack for Mexican Small Business

**Requirements:** low price, excellent output quality, security, reliability. Target: small business in Mexico, not enterprise.

**Recommended Stack (based on market analysis):**
- WhatsApp API: Meta Cloud API (free tier – 1,000 free conversations/month, then ~$0.0458 USD per business-initiated conversation in Mexico). Avoid BSPs like Twilio unless SLA needed.
- Backend: Python + FastAPI (lightweight, async, cheap hosting).
- AI/LLM: Claude Haiku (cost/performance) or GPT-4o-mini; both have strong Spanish capabilities.
- Hosting: Railway or Fly.io (low cost, simpler than AWS/GCP). For Mexican data residency, GCP `us-central1` or `northamerica-northeast1` is acceptable.
- Database: Supabase (PostgreSQL, free tier generous) RLS is mandatory. Sensitive customer data must be encrypted. Supabase's DPA must be signed. Must keep clear records of security practices.
- Session state: Upstash Redis (serverless, essentially free at low volume).

**Why avoid alternatives:**
- Twilio → adds $0.005–0.015/message markup.
- Dialogflow/IBM Watson → overkill, expensive, weaker Spanish nuance.
- AWS Lambda → complex cold starts.
- Node.js → Python has better LLM tooling.

**Mexico‑specific constraint:** Meta requires a verified Facebook Business Manager (1–3 weeks, may need RFC tax ID or utility bill). Build this into launch timeline.

**Cost estimate for ~500 users/month:**
- Meta API: $0–5 (within free tier)
- Railway: ~$5
- Supabase: Free
- Upstash Redis: Free
- Claude Haiku: $1–3
- **Total: $6–13 USD/month**

**Assumed reliability level:** Acceptable for non‑critical bots. For 99.9% uptime, add BSP with SLA and persistent Redis (cost increases).

## Task: Develop a Technical Plan for Two WhatsApp Chatbot Versions

**Reliability requirement to include in both plans:**
> Add a dead‑letter queue or retry mechanism for Meta webhooks. Meta expects a `200 OK` within ~15 seconds. If your LLM call times out, Meta may block the webhook temporarily. Solution: acknowledge immediately (`202 Accepted`), process asynchronously, and send the reply via the `/messages` endpoint later.

**Your task:**

Create two separate development plans – one for a **Basic Chatbot** and one for an **Advanced (AI‑powered) Chatbot** – using the recommended stack (Meta Cloud API, Python + FastAPI, Supabase, Upstash Redis, Railway/Fly.io, Claude Haiku or GPT‑4o‑mini).

For each plan, include:

1. **Core features** (what the bot can do)
2. **System architecture diagram** (described in text or ASCII)
3. **Step‑by‑step implementation phases** (from sandbox setup to production)
4. **How to implement the reliability improvement** (202 Accepted + async reply + dead‑letter queue/retry)
5. **Estimated development time** (person‑days or weeks)
6. **Estimated monthly operating cost** at 500 users/month (breakdown)
7. **Key Mexico‑specific considerations** (Business Manager verification, Spanish language nuance, data privacy)

**Basic Chatbot definition:**
- Rule‑based or simple keyword matching (no LLM, or optional minimal LLM fallback)
- Handles FAQs, store hours, basic lead collection
- No persistent conversation memory beyond current session

**Advanced Chatbot definition:**
- Uses LLM (Claude Haiku or GPT‑4o‑mini) for natural conversation
- Integrates with a CRM or external API (e.g., order status, appointment booking)
- Maintains long‑term user context and conversation history in Supabase
- Includes the async webhook pattern with dead‑letter queue

**Output format:** Use clear headings for Basic vs. Advanced. For each plan, present the 7 points as a numbered list.

**Optional:** Add a recommendation on which plan to start with for a Mexican small business on a tight budget.

## Planning Protocol (You Must Follow These Steps)

You will go through **exactly 5 steps**. After each step, you will wait for the user to authorize moving to the next step. Do not combine steps or skip ahead.

### Step 1: Clarify & Bound the Requirements
Ask the user these 4 questions (and only these):
1. What are the top 3 use cases? (e.g., FAQs, lead capture, order status)
2. Is an LLM required, or is rule‑based sufficient?
3. What is the maximum monthly conversation volume? (affects Meta tier and hosting)
4. Does the business already have a verified Facebook Business Manager?

**Output:** List the questions. Wait for answers.

**Gate:** Do not proceed to Step 2 until the user answers or explicitly says “use your best guess.”

### Step 2: Propose a High‑Level Architecture (No Code)
Write ≤10 bullet points describing:
- Webhook flow: 202 Accepted → queue → async worker → reply via /messages
- Where the dead‑letter queue lives (e.g., Redis list or Supabase table)
- Retry logic (max retries, backoff)
- Data flow in text (User → Meta → Webhook → Queue → Worker → Meta)

**Output:** Architecture description. Then ask: “Please approve this architecture or request changes.”

**Gate:** Do not proceed until user says “approved” or provides modifications.

### Step 3: Break Work into Small, Verifiable Milestones
List milestones where each produces a testable, non‑AI output. Example structure:
1. Sandbox webhook that returns 202 and logs.
2. Async worker that sends a fixed reply via Meta API.
3. Dead‑letter queue with retry (store failed messages, retry 3x).
4. Single rule‑based flow (e.g., keyword “hours”).
5. (If LLM) Integrate LLM with timeout fallback.

**Output:** Milestone list with clear “done” criteria for each. Then ask: “Please approve this milestone sequence.”

**Gate:** Wait for approval.

### Step 4: Write Pseudo‑Code for the First Milestone Only
Write pseudo‑code (not full runnable code) showing:
- The webhook endpoint structure (FastAPI)
- How it returns 202 and pushes to queue
- The async worker skeleton
- Signature verification check (X-Hub-Signature-256)

**Output:** Pseudo‑code. Then ask: “Does this match your expected behavior? May I proceed to implement milestone 1 in real code?”

**Gate:** Wait for explicit “yes, implement milestone 1.”

### Step 5: Implement Incrementally with Validation After Each Milestone
You will implement one milestone at a time. After completing each milestone, you will run a validation checklist:
- Does the webhook return 202 within 15ms?
- Is the signature verification header checked?
- (For queue milestones) Does the dead‑letter queue retry and log failures?

**Output:** After each milestone, present the validation results and ask: “Milestone X complete. Please review validation. Move to milestone Y?”

**Gate:** Do not proceed to the next milestone without authorization.

## Final Constraint
If at any point you are unsure about a requirement, ask a single clarifying question. Do not invent missing details.

Begin with Step 1.

### IMPLEMENTATION (after all gates are approved)

Write the complete, working implementation for the current milestone. Follow all rules below.

#### Code Quality Requirements

1. **Type hints** – Use Python type hints for all function parameters and return values. Include `Optional`, `Union`, `List`, `Dict` from `typing` where needed.

2. **Error handling** – Handle at least these edge cases:
   - Missing or malformed JSON body in webhook
   - Invalid signature (log and return 401, don’t crash)
   - Meta API rate limits (catch 429, retry with exponential backoff)
   - Database or Redis connection failures (fallback to local queue or log)
   - LLM timeout (fallback to rule‑based reply or “I’m having trouble, please try again”)

3. **Logging** – Use Python’s `logging` module with different levels (INFO, WARNING, ERROR). Do not use `print()` except for quick debugging comments that must be removed.

4. **Comments** – Write **human‑friendly, explanatory comments** on any non‑obvious logic. Avoid obvious comments like `# increment counter`. Instead explain *why* something is done (e.g., `# Return 202 immediately to prevent Meta from retrying`).

5. **Naming conventions** – Use descriptive, self‑documenting names:
   - Functions: `verify_whatsapp_signature`, `send_reply_via_meta`, `queue_failed_message`
   - Variables: `user_message_text`, `retry_count`, `webhook_received_at`
   - Constants: `MAX_RETRIES = 3`, `META_API_TIMEOUT_SECONDS = 12`

#### Human‑Language Style (No AI‑Tells)

Write code and comments as a competent human developer would. Follow these rules strictly:

- **No exclamation points** – Never use `!` in comments or strings unless quoting an external error message.
- **No overly enthusiastic language** – Avoid words like “amazing”, “powerful”, “simply”, “just”, “of course”, “obviously”. Be neutral and factual.
- **Use contractions** – Write `doesn’t`, `it’s`, `won’t`, `can’t`, `we’ll` instead of `does not`, `it is`, `will not`, `cannot`, `we will`.
- **No emojis** – Not in comments, log messages, or reply strings. Zero exceptions.
- **No exclamatory punctuation** – Use periods, not multiple punctuation marks (e.g., `...` or `?!`).
- **Keep tone modest** – Instead of “This function efficiently handles everything”, write “This function handles the webhook verification.”

#### Git Commit Message Style (Optional Reference)

If you are later asked to suggest a commit message, follow these conventions: imperative tense, lowercase after first word, no trailing period, explain what/why not how, avoid “This commit implements…”. Example: `add retry logic for rate limits`

#### Avoiding “AI Flags” in Code

- **Vary code structure** – Don’t write every function with exactly the same indentation and blank‑line pattern.
- **Include small imperfections** – A rare `# TODO: clean this up later` or an extra blank line before a return is fine.
- **Mix condition styles** – Use `if not x` sometimes, `if x is False` rarely, but prefer the natural `if not x`.
- **Don’t over‑explain trivial lines** – No comment for `return True` or `counter += 1`.
- **Skip unnecessary try/except** – Only catch exceptions that are likely to happen in production.

#### Self‑Check (Run This After Writing Code)

Before presenting the validation checklist, mentally run through these checks and note any failures. If any check fails, revise the code before showing it.

**1. Correctness – does it solve the exact problem?**
- Does the code satisfy the milestone’s stated goal? (Yes / No)
- Does it handle the primary success path without crashing?

**2. Edge cases handled (test mentally)**
- What happens if the webhook receives an empty body?
- What if the Meta API returns a 429 (rate limit)?
- What if the database is temporarily unreachable?
- What if the LLM call times out (if applicable)?
- What if the user sends an unsupported message type (image, location)?

**3. Input/output type matching**
- Are all function arguments annotated with type hints?
- Does every function return the promised type (or `None`)?
- Does the webhook return a `Response` object with status code?
- Does the queue worker expect the correct message structure?

**4. Simplicity check – could this be simpler or more efficient?**
- Ask: “Is there a built‑in library or language feature that replaces 10+ lines of my code?”
- Ask: “Am I over‑engineering for the current milestone?” (e.g., adding a full state machine when a simple `if` works)
- Ask: “Does this solution use more memory or CPU than necessary for 500 users/month?”
- If yes to any, note a **simpler alternative** (do not rewrite unless it’s a major issue).

**5. No hidden assumptions**
- Does the code assume a certain environment variable is always set? (Should check `os.getenv` with default or explicit error.)
- Does it assume the Redis server is always up? (Should have a fallback or log.)
- Does it assume the Meta API endpoint never changes? (Acceptable, but note it.)

#### Output Format for Each Milestone

For each completed milestone, output in this exact order:

1. **Self‑check results** – a short paragraph or bullet list.
2. **Code block** – with `# BEGIN CODE` and `# END CODE` inside triple backticks (`python`).
3. **Validation checklist** – bullet list of passed/failed items.
4. **Gate question** – e.g., “Milestone X complete. Move to milestone Y?”

**Example (follows the order above):**

**Self‑check results:**  
- Correctness: passes.  
- Edge cases: handles missing body, rate limits.  
- Simplicity: uses built‑in `json.loads`, no over‑engineering.  

```python
# BEGIN CODE
import os
from fastapi import FastAPI, Request, Response

app = FastAPI()

@app.post("/webhook")
async def whatsapp_webhook(request: Request) -> Response:
    # Return 202 immediately to avoid Meta retries
    return Response(status_code=202)
# END CODE