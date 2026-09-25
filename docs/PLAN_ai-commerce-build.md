# Implementation Plan — AI Commerce Build

**Title:** AI commerce web app: storefront, checkout, returns, and text/voice assistant

**Source:** `ai-commerce-build-handoff.zip` (8 files), plus the review and research done on 2026-09-25

**Owner:** Taro Serigano (with a coding agent such as Claude Opus 5.5)

**Estimated effort:** about 40 working days for one developer driving a coding agent. This is a rough guess: agent speed varies a lot, and the original plan assumed 10–14 weeks for 3–5 engineers.

**Target success metric:** A new developer can run the full demo from the README: buy a product with Stripe test mode, return it by text or voice, and see the correct refund status. The acceptance test suite passes, with zero unauthorized or duplicate refunds in the adversarial tests.

---

## Decisions made by Taro (2026-09-25)

These override the handoff docs where they differ.

| Topic | Decision |
|---|---|
| Vector database | **Pinecone only, no exceptions.** It holds both policy/help text and product "meaning" vectors. |
| Local development | Uses a **real Pinecone account**. Each developer gets their own namespace (a separate folder inside the index). |
| Tests and Pinecone | **Pure unit tests only** may use a small stand-in at the Pinecone adapter boundary, so they run fast and offline. The running app (local, CI, cloud), all integration tests, all evals, and the demo use **real Pinecone**. The stand-in lives only in `tests/unit/` and can't be turned on by any app setting. |
| Product search | OpenSearch handles keywords, filters, and facets. Pinecone handles meaning-based search. The commerce API merges the two result lists. |
| Sales tax | **Stripe Tax** calculates tax inside Stripe Checkout. Our server does not calculate its own tax. |
| treg | **Dropped.** Its data is other retailers' listings, which the README forbids. |
| Monitoring | **OpenTelemetry (OTel) everywhere**: traces, metrics, and logs from the browser to Stripe, built from Milestone 1. See "Evaluation and monitoring system" below. |
| Evals | **A strong, layered eval system**, including **Ragas** for policy-answer quality. It runs on every pull request, nightly, and on live pilot traffic. |
| Sign-in | **Sign-in comes first.** Every visitor lands on the sign-in screen and must sign in (or create an account) before using any part of the site: browsing, search, cart, help, or the assistant. The sign-in screen is the first thing built. |
| Guest checkout | **None.** There is no guest access at all, so there is also no guest cart and no cart merge on sign-in. |
| AWS account | **Taro's personal AWS account only, never a RealPage account.** Use a named AWS CLI profile (for example `personal`). Terraform locks itself to that account ID with `allowed_account_ids`, so it refuses to run against any other account. Local development needs no AWS at all. |
| Temporal in Milestone 2 | **Used**, as the handoff says. A Temporal checkout workflow handles the wait for payment, reservation expiry, and reconciliation. |

## Claude's proposed defaults (not yet approved; change any)

| Topic | Default | Differs from the handoff? |
|---|---|---|
| Jev | Kept as an optional experiment in Milestone 5, behind a feature flag | No |
| Payments | Stripe **Checkout Sessions** (Stripe's recommended option), cards only, automatic capture in the sandbox | Picks one of the options the handoff left open |
| Refund destination | The original payment method only; no store credit in the pilot | Removes the idea of several refund methods |
| Accessibility | WCAG 2.2 AA (the standard web accessibility checklist) | Adds a level the handoff didn't name |
| Sandbox return policy | The table in Step 1 | Adds rules the handoff left to the agent |

---

## Goal

Build a working, portfolio-quality online store with an AI helper. A shopper can browse, buy with a Stripe test card, and track their order.

The shopper can also ask the assistant, by typing or speaking, to return an item. The backend decides whether the return is allowed and how much to refund. The customer confirms by clicking a button, or by a clearly checked spoken "yes".

The AI never confirms a return or refund on its own.

## Non-goals

- A Walmart clone, or copying any retailer's designs or data.
- Real payments, real refunds, or a live launch.
- Guest access of any kind (browsing, cart, or checkout without signing in).
- Search-engine visibility (SEO). Product pages sit behind sign-in, so search engines can't index them.
- Marketplace sellers, loyalty, store pickup, or international tax.
- Native mobile apps. The site is responsive web only.
- Kubernetes, Kafka, or many separate microservices.
- treg in any form.
- Any vector store other than Pinecone.

---

## Pre-flight Checklist

- [ ] New project folder with **no trailing space and no "Walmart" in the name**. Example: `~/Documents/DEVO/ai-commerce/`.
- [ ] `git init` done, and the handoff docs unzipped into `docs/handoff/`.
- [ ] Docker Desktop running.
- [ ] Accounts and test keys ready, stored only in `.env`, never in Git:
  - [ ] Stripe **test mode**, plus the Stripe CLI for local webhooks
  - [ ] Stripe Tax turned on in test mode, with one test state registration (otherwise tax shows $0)
  - [ ] Pinecone project and API key
  - [ ] Embeddings: Pinecone's hosted embedding models if available on the plan, otherwise one embeddings provider
  - [ ] LLM provider key, starting with Claude Sonnet 5
  - [ ] LiveKit Cloud, Deepgram, and Cartesia (needed only in Step 6)
  - [ ] **Personal** AWS account (needed only for the cloud work in Step 7):
    - [ ] Install the AWS CLI.
    - [ ] Create a profile named `personal`, using IAM Identity Center or an IAM user with MFA. Don't use long-lived root keys.
    - [ ] Record the 12-digit account ID for Terraform's `allowed_account_ids`.
    - [ ] Set a billing alarm (for example, $25 per month) before creating anything.

---

## Evaluation and monitoring system

This section applies across every step. Each step below adds its own part and has an exit check for it.

### A. Monitoring with OpenTelemetry

**The idea:** every customer action becomes one **trace**: a timeline of every service call it caused. We can follow a single refund from the click, through the API, Temporal, and Stripe, and back.

| Part | What it does |
|---|---|
| **Instrumentation** | Every service sends OTel data. The confirmed packages (checked 2026-09-25) are:<br>• **Next.js:** `instrumentation.ts` with `registerOTel` from `@vercel/otel`.<br>• **Python:** `opentelemetry-instrumentation-fastapi`, `-sqlalchemy`, `-httpx`, and `-redis`.<br>• **Temporal:** `temporalio.contrib.opentelemetry.TracingInterceptor`.<br>• **LiveKit:** `livekit.agents.telemetry.set_tracer_provider(..., allow_pii=False)`. This covers traces only.<br>• **LangGraph/LangChain:** `langsmith[otel]` with `LANGSMITH_OTEL_ENABLED=true`.<br>• **Browser:** the `web-vitals` library (`onLCP`, `onINP`) for page-speed scores. The OTel browser trace SDK is still experimental, so keep browser tracing minimal. |
| **One trace per action** | Standard W3C trace headers pass between services. The trace ID also goes into outbox events and Stripe `metadata`, so webhooks join the same story. |
| **Collector** | One OTel Collector (the contrib build) receives everything. Its redaction, attributes, and transform processors **remove personal data**: emails, names, addresses, and card-like numbers. Its tail-sampling processor **keeps 100%** of errors, money operations, and voice confirmations, and samples 10% of normal traffic. |
| **Where data goes** | **Locally:** the `grafana/otel-lgtm` Docker image, one container with Grafana, Tempo (traces), Loki (logs), and Prometheus (metrics). It is for development only.<br>**Cloud:** AWS's collector (ADOT) runs beside each ECS Fargate task and sends traces to X-Ray and metrics to CloudWatch.<br>**AI traces** also go to LangSmith.<br>Because everything flows through the collector, we can change tools without changing code. |
| **Shared labels on every span** | `customer_hash` (never the raw ID), `conversation_id`, `operation_id`, `quote_id`, `order_id`, `policy_version`, `model_id`, `prompt_version`. LLM spans use the OTel GenAI names (`gen_ai.*`). These names are still in "Development" status and may change, so pin the version. |
| **Logs** | JSON logs, each with a trace ID, and no personal data. |

**Metrics to collect:**

| Group | Metrics |
|---|---|
| Shopping | Search speed (p50/p95), zero-result searches, add-to-cart rate, index lag (time from a product change to searchable) |
| Money | Checkouts started, paid, failed, and expired; webhook delay; checkouts stuck in reconciliation; refunds by state; **duplicate requests blocked**; **ownership denials** |
| AI | Tokens and **cost per conversation**, tool calls per turn, tool errors, handoff rate, answers with no citation, cap hits |
| Voice | Time per stage (speech-to-text, first LLM token, first audio), end-of-speech to first audio, interruption stop time, session minutes, false-confirmation blocks |
| Platform | Request rate, errors, and duration per endpoint; outbox lag; Temporal workflow failures; Pinecone, OpenSearch, and LLM latency and errors |

**Dashboards (4):** Shopping and checkout · Returns and refunds (money) · Assistant quality and cost · Voice.

**Alerts:**

| Level | Fires when |
|---|---|
| **Page now** | Our refund records don't match Stripe's. A duplicate refund is detected. Ownership denials suddenly jump (a possible attack). Webhook signature failures jump. |
| **Warn** | Search p95 over 300 ms. Commerce availability below 99.9%. Outbox lag over 60 s. A checkout stuck over 15 min. LLM spend per hour over budget. Voice p95 over 2 s. Handoff rate jumps. Live eval scores drop (see B). |

**Money audit trail:** every refund links quote → approval → return → refund attempt → Stripe refund ID → trace ID. A daily reconciliation job compares our records with Stripe and reports any gap.

### B. Evaluation system (including Ragas)

**The idea:** test the AI the way we test code, with fixed datasets, automatic scores, and gates that block bad changes. Safety checks use plain code, never an AI judge.

| Layer | What it checks | Tool | When it runs | Gate |
|---|---|---|---|---|
| **1. Safety and behavior checks** | Right tools in the right order; no tool used for another customer; **no confirmation without a real approval**; amounts match the backend exactly; each citation points to a real policy version | pytest with `langsmith[pytest]` (`@pytest.mark.langsmith`), plus `agentevals` trajectory match (strict, unordered, subset, or superset modes) | Every pull request (fast set of about 30), and nightly (full set) | **Hard gate: 100%** |
| **2. Policy answer quality (Ragas)** | `Faithfulness` (the answer sticks to the retrieved policy), `AnswerRelevancy`, `ContextPrecision` (retrieved text was useful), `ContextRecall` (the right section was found), and `NoiseSensitivity` (irrelevant text doesn't mislead it) | **Ragas 0.4.x**, with Claude as the judge | Nightly, and on any change to prompts, embeddings, or policy text | Must not drop more than a set margin below the baseline |
| **3. Search quality** | Policies: did the right section come back in the top 5 from Pinecone? Products: are the top 10 results good for about 100 labeled queries? | Our own script | On any index, embedding, or ranking change | Must not drop below the baseline |
| **4. Conversation quality** | Did the assistant finish the task? Pick the right item? Stay on topic? Hand off when it should? | Ragas agent metrics (`ToolCallAccuracy`, `ToolCallF1`, `AgentGoalAccuracyWithReference`, `TopicAdherence`) plus an LLM judge | Nightly | Must not drop below the baseline |
| **5. Voice audio replay** | Real recordings (accents, noise, pauses, barge-in, "mm-hmm") run through the real speech-to-text → graph → speech pipeline | Our own runner | Weekly and before a release (it costs money) | **False confirmations = 0**; latency is reported per stage |
| **6. Red team** | Prompt injection in product and policy text; asking for another customer's data; "refund me $500"; jailbreak attempts | Scripted attack set | Every release | **Hard gate: 0 unsafe actions** |
| **7. Live evals** | Sample pilot conversations (with personal data removed) and score them. Also collect thumbs up and down. | LangSmith online evaluation (an LLM judge on sampled traces). Ragas can't read LangSmith traces directly, so a scheduled job pulls sampled traces and runs Ragas `Faithfulness` on them. The job writes the scores back to LangSmith. | Continuously (LangSmith); hourly (the Ragas job) | Alert when scores drop |

**Rules that keep evals honest:**

- **Two datasets.** A *dev* set for tuning, and a *held-out* test set that is never used for tuning. This stops us from "teaching to the test".
- **Check the judge.** Before an AI-judge score becomes a gate, compare it with human labels on 50 examples, and record how often they agree.
- **No invented targets.** Soft gates start from a measured baseline, as the handoff requires.
- **Any change to the prompt, model, tools, embeddings, or policy text** runs the full suite and a side-by-side comparison with the baseline. Prompts are versioned in the repo, and their version is added to every trace.
- **The loop.** A failed live conversation goes to a review queue. After review, it becomes a new eval scenario.
- **Privacy.** Datasets use synthetic customers only. Personal data is removed from any pilot sample before an outside judge sees it.

**How to wire Ragas** (checked 2026-09-25; the Ragas API changed in version 0.4):

- Pin `ragas==0.4.*` and install the `anthropic` package separately, because Ragas does not include it.
- Use the **new** API: metrics from `ragas.metrics.collections`, the `@experiment` decorator, and `metric.ascore(...)`, which returns a `MetricResult`.
- **Do not use the old `evaluate()` function.** It is deprecated and throws a TypeError with the new metrics (Ragas issue #2624).
- Use Claude as the judge with `llm_factory(model, provider="anthropic", client=Anthropic())`.
- Turn LangGraph conversations into Ragas format with `ragas.integrations.langgraph.convert_to_ragas_messages`.
- Draft the first policy questions with Ragas's `TestsetGenerator` from the policy documents. **A person reviews every question** before it enters the dataset.
- Keep all Ragas calls in one module (`evals/ragas_runner.py`), so a future API change touches one file.
- `openevals` (groundedness and retrieval relevance checks) is a backup if a Ragas metric turns out unreliable.

**Dataset files** (in `evals/`):

| File | Holds |
|---|---|
| `scenarios/*.jsonl` | One conversation per line: `id`, `category`, `input` or `audio_file`, `customer_id`, `expected_intent`, `expected_tools`, `must_not` (for example, `confirm_without_approval`), `expected_handoff`, `reference_answer`, `reference_policy_sections`, `split` (dev or test), `tags` |
| `policy_qa.jsonl` | About 100 policy questions with reference answers and the correct section IDs (for Ragas and the search checks) |
| `search_queries.jsonl` | About 100 product queries with graded good results |
| `audio/` | Recorded audio for voice replay, with consent and no real customer data |
| `redteam/` | Attack scripts |

---

## Step-by-Step Plan

### Step 0 — Set up the project (~0.5d)

**Why:** The current folder name ends in a space, which breaks scripts and Docker paths. The folder is also not a Git repo.

1. Create the new folder (see Pre-flight) and run `git init`.
2. Unzip the handoff into `docs/handoff/`.
3. Add a `CLAUDE.md` whose only line is `@docs/handoff/AGENTS.md`, so Claude Code loads the agent rules.
4. Copy this plan into `docs/PLAN_ai-commerce-build.md`.
5. Make the first commit.

**Exit criteria:** `git log` shows one commit containing the docs, `CLAUDE.md`, and this plan.

### Step 1 — Fix the handoff docs before the agent reads them (~0.5d)

**Why:** The coding agent follows the docs exactly. The gaps found in the review must be closed first, or the agent will guess.

Make these edits in `docs/handoff/`:

| File | Edit |
|---|---|
| `TECHNICAL_SPEC.md` — tools list | Remove `confirm_return` from the tools the model can call. Add `propose_return(quote_id)`. It pauses the conversation (LangGraph `interrupt()`) and shows the quote card. Only the browser's Confirm button, or the app's voice-confirmation step, may call `POST /v1/returns/confirm`. The confirmation token is **never** put in the model's context. The tool result the model sees holds only a quote summary. The browser gets the token from its own signed-in call, `GET /v1/returns/quotes/{id}`, and never from conversation or graph state. |
| `TECHNICAL_SPEC.md` — stack table | Policy retrieval: Pinecone, real account in every environment, one namespace per environment. Product discovery: OpenSearch for keywords and filters, and Pinecone for meaning. Remove "local deterministic substitute". |
| `TECHNICAL_SPEC.md` — payment | Use Stripe Checkout Sessions with `automatic_tax`. The checkout quote shows "tax calculated at payment". After payment, save each line's tax and discount from Stripe's line items. |
| `TECHNICAL_SPEC.md` — states | Split checkout-attempt states from order states. The order row is created only after a verified `checkout.session.completed` (or the async-success event). Add Stripe's refund states `REQUIRES_ACTION` and `CANCELED`. A `SUCCEEDED` refund can later fail, so keep listening for `refund.updated` and `refund.failed`. |
| `TECHNICAL_SPEC.md` — retries | Before **every** refund retry, check Stripe first. Stripe forgets idempotency keys after 24 hours. |
| `TECHNICAL_SPEC.md` — local ops | Outbox publisher: a polling worker using `SELECT … FOR UPDATE SKIP LOCKED`. It feeds OpenSearch and Pinecone locally. EventBridge and SQS are used in the cloud only. |
| `PRODUCT_REQUIREMENTS.md` | Add the sandbox return policy table (below). **Remove the Guest role**: the whole site needs sign-in, with no guest cart, cart merge, or guest order check. Only these pages are public: sign-in, create account, forgot password, privacy, terms, and the accessibility statement. Remove store credit. Target WCAG 2.2 AA. |
| `TECHNICAL_SPEC.md` — API | `GET /v1/products` and `GET /v1/products/{id}` now need sign-in, like every other route. The only routes without sign-in are the auth routes, `/healthz`, and the Stripe webhook (which checks its signature instead). |
| `IMPLEMENTATION_PLAN.md` | Milestone 1 starts with the **sign-in screen and sign-in wall**. Milestone 2 **uses Temporal** for the checkout workflow. Milestone 1's exit check adds "CI runs lint, type checks, and tests". Admin catalog editing moves to Milestone 5 (a basic form). |
| `ACCEPTANCE_AND_EVALS.md` | Copy in the "Evaluation and monitoring system" section from this plan: the 7 eval layers and their gates, Ragas, the dataset files, the alerts, and the money audit trail. Accent and noise cases need recorded audio in `evals/audio/`. Add per-user caps on voice minutes, sessions, and LLM tokens. |
| `TECHNICAL_SPEC.md` — telemetry row | OpenTelemetry in every service, one OTel Collector (removes personal data, keeps all money and error traces), Grafana locally, CloudWatch/X-Ray in the cloud, and LangSmith for AI traces. |
| `ecommerce-architecture-plan.md` | Delete line 196 (the Postgres/OpenSearch vector alternative). Delete every treg section and S22. Fix citations: the replay warning is in S7, not S6. Change S8 to `https://docs.temporal.io/activity-definition`. |
| `README.md`, `MASTER_PROMPT.md`, `AGENTS.md` | Remove treg mentions. |

**Sandbox return policy** (add this to the requirements so tests have fixed rules):

| Rule | Sandbox value |
|---|---|
| Return window | 30 days after delivery |
| Not returnable | "Final sale" category (for example, gift cards) |
| Needs inspection | "Electronics" category |
| Instant refund | Items under $25 that don't need inspection: refund when the carrier scans the package |
| Other refunds | Refund after warehouse inspection |
| Restocking fee | None |
| Shipping refund | Only when the whole order is returned |
| Partial returns | Refund = the line's saved price − saved discount + saved tax, per unit |
| Rounding | Split cents evenly across units; the **last** unit returned takes any remainder. For example, a $10.00 discount over 3 units is 333, 333, 334 cents. Refunds for a line can never add up to more than that line's saved total. |

**Exit criteria:** `grep -riE "treg|pgvector|PostgreSQL vector search|deterministic substitute" docs/handoff` finds nothing. The edits are committed.

### Step 2 — Milestone 1: sign-in and shopping foundation (~6.5d)

**Why:** Nobody can use the site without signing in, so sign-in is built first. Everything else needs a working catalog, cart, and search.

1. Create the monorepo: `apps/web`, `services/commerce`, `services/assistant`, `services/voice`, `workers/workflows`, `packages/contracts`, `infra`, `tests`. Pin all versions in lockfiles.
2. Set up Docker Compose with PostgreSQL, Redis, OpenSearch, and the Temporal dev server. Pinecone is the real cloud service.
3. **Build the sign-in screen first:**
   - Pages: sign-in, create account, forgot password, and sign-out.
   - **Local:** a development sign-in adapter with seeded accounts: two customers (`alice`, `bob`) and one staff user. The screen shows a clear "Development sign-in" label.
   - **Cloud:** the same screen hands off to Amazon Cognito through the production OIDC interface (OIDC is the standard sign-in protocol).
   - Sessions use a secure, HttpOnly, `SameSite=Lax` cookie with a set expiry. Sign-out ends the session on the server too.
   - The sign-in form works with password managers and allows pasting, as WCAG 2.2 "Accessible Authentication" (3.3.8) requires. It has clear error messages, and the same "wrong email or password" message whether or not the account exists.
   - Basic rate limit on sign-in attempts from day one.
4. **Build the sign-in wall:**
   - Next.js `proxy.ts` sends any signed-out visitor to `/sign-in?returnTo=<page>`. After sign-in, it returns them to that page.
   - `returnTo` must be a path on our own site, so it can't be used to send people to another site.
   - The public allowlist is: sign-in, create account, forgot password, privacy, terms, the accessibility statement, static files, `/healthz`, and the Stripe webhook.
   - **The redirect is only for convenience.** Every FastAPI route still checks the session itself and returns 401 when signed out. Never rely on `proxy.ts` alone for security.
5. Build the FastAPI catalog, cart, and orders modules. Add Alembic migrations. Seed about 50 original products. Images are generated placeholders stored in the repo; never link to images on other websites.
6. Build the outbox polling worker. It indexes products into OpenSearch (keywords) and Pinecone (namespace `products-<env>`).
7. Build hybrid search: run OpenSearch and Pinecone, then merge with reciprocal rank fusion (a standard way to combine two ranked lists). Always load current price and stock from PostgreSQL.
8. Add a `make reindex` command that rebuilds OpenSearch and Pinecone from PostgreSQL.
9. Generate the TypeScript API client from FastAPI's OpenAPI schema.
10. Build the Next.js pages behind the wall: home, category, search, product, and cart.
11. Add GitHub Actions CI that runs lint, type checks, and unit tests.
12. **Monitoring foundation:**
    - Add the OTel Collector and the Grafana stack to Docker Compose.
    - Instrument Next.js, FastAPI, SQLAlchemy, httpx, Redis, and the outbox worker.
    - Add the shared span labels and JSON logs with trace IDs.
    - Set up personal-data removal in the collector.
    - Build the "Shopping" dashboard, plus the index-lag and search-speed metrics.
    - Add sign-in metrics: successes, failures, and rate-limit hits. Never record passwords or raw emails.
13. **Search evals v1:** write `evals/search_queries.jsonl` (about 50 queries to start) and a script that scores product search quality.

**Exit criteria:**

- A fresh clone starts with the README commands.
- **Sign-in wall** (Playwright tests):
  - A signed-out visitor who opens `/`, a product page, or `/cart` lands on the sign-in screen.
  - After signing in, they return to the page they asked for.
  - A `returnTo` pointing to another site is ignored.
  - Calling any private API with no session returns 401, even when `proxy.ts` is bypassed.
  - Signing out, then pressing Back, shows no private data.
- Searching "wireless headphones" and "waterproof shoes for rain" both return sensible results.
- Price and stock always come from PostgreSQL.
- Bob gets a 404 on Alice's cart.
- CI is green.
- One search shows up in Grafana as **one trace**: browser → Next.js → commerce API → OpenSearch, Pinecone, and PostgreSQL.
- A test sends a fake email address through a request and proves it never appears in the exported traces or logs.
- The search-quality baseline is recorded.

### Step 3 — Milestone 2: checkout and orders (~6.5d)

**Why:** Returns need a real paid order.

1. Build `POST /v1/checkout/quote`. The server checks price and stock, and shows "tax calculated at payment".
2. Build `POST /v1/checkout/start`. It uses an idempotency key and starts a **Temporal checkout workflow**, whose workflow ID is the checkout attempt ID. A double-click therefore finds the same workflow and can't start a second one.
3. **Temporal checkout workflow:**
   - CREATED → reserve stock (activity) → STOCK_RESERVED → create the Stripe Checkout Session with `automatic_tax` (activity) → PAYMENT_PENDING → wait for a "payment result" signal, or the timer.
   - Paid: → PAID → create the order (activity) → FULFILLING (mock fulfillment activity).
   - Failed, expired, or canceled: release the stock (activity) → FAILED, EXPIRED, or CANCELED.
   - If the timer fires with no signal, first ask Stripe for the session's real status (reconciliation), and only then expire it.
   - Set the session's `expires_at` to match the workflow timer and the stock hold, so payment can't succeed after stock is released. Stripe's session default is 24 hours; check its allowed minimum, believed to be 30 minutes, and use that as the hold length.
   - Every activity is safe to repeat and uses a deterministic operation ID. The workflow itself calls no outside services, as Temporal requires.
4. Build the webhook route. It verifies the signature on the raw request body, saves and deduplicates the event, replies quickly, and then **signals the checkout workflow**. It handles:
   - `checkout.session.completed`
   - `checkout.session.async_payment_succeeded`
   - `checkout.session.async_payment_failed`
   - `checkout.session.expired`
5. Create the order only from verified payment evidence: `checkout.session.completed` with `payment_status = paid`, or the async-success event. Save each line's `amount_discount`, `amount_tax`, and `amount_total` from the Stripe line items.
6. Add a reconciliation schedule (a Temporal schedule). Every 15 minutes it checks Stripe for checkout workflows still waiting, because sandbox retries failed webhooks only 3 times.
7. Build the order list and order detail pages. Show a "Sandbox payments" banner.
8. Document local webhooks: `stripe listen --forward-to localhost:8000/v1/webhooks/stripe`.
9. Test with Temporal's time-skipping test environment, so a 30-minute hold runs in seconds.
10. **Monitoring:**
    - Add Temporal's OTel tracing interceptor, so workflows and activities join the trace.
    - Add spans for Stripe calls.
    - Put the trace ID and checkout attempt ID into Stripe `metadata`, so the webhook joins the same trace.
    - Add the money metrics: checkouts by state, webhook delay, stuck checkouts, duplicates blocked, and ownership denials.
    - Build the "Checkout" part of the money dashboard.
    - Add alerts for stuck checkouts and webhook signature failures.

**Exit criteria:** Tests pass for:

- a successful payment;
- a declined card;
- a double-clicked checkout (one attempt only);
- a changed price (a new quote is required);
- an expired reservation (stock is released);
- duplicate and out-of-order webhooks;
- a webhook that never arrives (the timer and reconciliation settle it correctly);
- a worker crash in the middle of checkout (the workflow resumes, with no double reservation and no double order);
- Bob reading Alice's order (denied).

An order is PAID only after verified Stripe evidence. Stripe shows non-zero tax for the test state.

One purchase shows as **one linked trace**, from the click to the webhook to the order. A double-click raises the "duplicates blocked" metric by 1.

### Step 4 — Milestone 3a: returns backend (~5.5d)

**Why:** This is the money-critical core. It must be correct before any AI touches it.

1. Add the policy versions table and a rules engine for the sandbox return policy. Save the policy version used on each order line.
2. Build `POST /v1/returns/quote`. It checks the owner, the delivery date, the category, and the quantity already returned. It returns the exact amount from the saved per-line numbers, an expiry of 15 minutes, and a quote hash (a fingerprint of the exact quote terms).
3. Build `POST /v1/returns/confirm`, called only by the browser or the voice-confirmation step. It needs the quote ID, a single-use confirmation token tied to the customer and the quote hash (stored only as a hash), and an idempotency key. It rechecks everything, then creates exactly one return.
4. Build the Temporal return workflow: label (mock) → wait for carrier scan → wait for inspection if needed → refund activity → follow the refund status.
5. Build the refund activity. It checks Stripe before every attempt and uses a deterministic operation ID. It tracks all Stripe refund states, including `requires_action`, `canceled`, and a later failure.
6. Confirm how Stripe Tax records **partial** refunds on Checkout payments. If Stripe needs a tax reversal, create one.
7. Add development-only endpoints, which are turned off unless `APP_ENV=dev`:
   - `POST /dev/orders/{id}/deliver`
   - `POST /dev/returns/{id}/carrier-scan`
   - `POST /dev/returns/{id}/inspect`
8. Add an injectable clock for business rules only. Do not fake time in Stripe signature checks, which allow only 5 minutes of drift. Use Temporal's time-skipping test environment.
9. Build a staff review queue page for exceptions.
10. **Monitoring:**
    - Reuse the Temporal tracing interceptor from Step 3 for the return workflow.
    - Build the money audit trail (quote → approval → return → refund → Stripe ID → trace ID).
    - Build the daily reconciliation job against Stripe.
    - Add the **page-now** alerts for a refund mismatch and a duplicate refund.
    - Finish the "Returns and refunds" dashboard.

**Exit criteria:** Tests pass for:

- one return per eligible line;
- an altered quote, an expired quote, and a reused token (all rejected);
- a double confirm (same return comes back);
- a quantity already returned;
- an excluded category and a window that has passed;
- Bob quoting Alice's order (denied);
- a worker crash after the refund call (no second refund);
- a refund that is pending, then fails.

The demo reaches "Refund SUCCEEDED" by using the dev endpoints.

One return shows as one trace across the API, the Temporal workflow, and Stripe, even with a multi-day wait simulated. A deliberately faked mismatch between our records and Stripe fires the page-now alert.

### Step 5 — Milestone 3b: text assistant and eval harness (~6d)

**Why:** It adds the AI layer on top of the backend proven in Step 4.

1. Build one LangGraph with three routes (general info, order status, return), using the PostgreSQL checkpointer.
2. Add typed tools: `list_my_orders`, `get_order_status`, `get_return_options`, `propose_return`, `search_products`, `get_product_details`, and `create_support_case`. The server adds the customer's identity to every call.
3. Make `propose_return` call `interrupt()`. The graph resumes only from the app after the confirm endpoint answers. Any side effect goes **after** the interrupt and is safe to repeat.
4. Load approved policy text into Pinecone (namespace `policies-<env>`), with version, effective dates, category, and section ID. Filter before searching. Show citations on screen.
5. Stream answers to the browser over SSE (server-sent events). Build the cards: product, order, return quote (with the Confirm button), status, citation, and handoff.
6. Treat product descriptions and policy text as data, never as instructions. Add prompt-injection test fixtures.
7. **AI monitoring:**
   - Send LangGraph and LangChain spans through OTel, using the `gen_ai.*` labels.
   - Also send AI traces to LangSmith.
   - Record tokens, cost per conversation, tool calls, tool errors, handoffs, and answers with no citation.
   - Keep prompts as versioned files, and add `prompt_version` to every trace.
   - Build the "Assistant quality and cost" dashboard.
8. **Eval harness v1:**
   - Write about 60 scenarios in `evals/scenarios/` (dev and held-out test splits).
   - Write `evals/policy_qa.jsonl` (about 50 questions to start).
   - Build eval layer 1, the safety checks: trajectory match, identity, no self-confirm, exact amounts, valid citations.
   - Build eval layer 2 with **Ragas 0.4.x**, using the new API (see "How to wire Ragas"): `Faithfulness`, `AnswerRelevancy`, `ContextPrecision`, `ContextRecall`, and `NoiseSensitivity`, with Claude as the judge.
   - Build eval layer 3 for policy search: is the right section in Pinecone's top 5?
   - Store the runs as LangSmith experiments.
9. **CI gate:** add `.github/workflows/evals.yml`.
   - A fast set of about 30 scenarios runs on every pull request that touches the assistant, prompts, tools, or policy text.
   - Any layer-1 failure blocks the merge.
   - The full suite plus Ragas runs nightly.

**Exit criteria:**

- "Return the headphones" with two matching items shows two cards and asks which one.
- An FAQ answer cites a policy version.
- Asking about a policy that doesn't exist leads to "I don't know" or a handoff.
- Bob can't reach Alice's conversation.
- A test proves the model's tool list has no confirm tool. It also proves no confirmation token appears in any model message, tool result, or saved graph checkpoint.
- The first Ragas and safety baseline is recorded (scores plus cost per eval run).
- A pull request that deliberately breaks a tool rule is **blocked** by the CI eval gate.
- One chat turn shows as one trace: browser → assistant → LLM → tool → commerce API → Pinecone.

### Step 6 — Milestone 4: voice (~6d)

**Why:** Voice is the headline feature. It reuses the graph and tools from Step 5.

1. Build `POST /v1/voice/token`: a short-lived, room-scoped LiveKit token.
2. Build the Python LiveKit worker with Deepgram Flux (`turn_detection="stt"`) and Cartesia pinned to `sonic-3.6-2026-08-27`. It shares the Step 5 graph.
3. Build the voice confirmation state, which the app controls, not the model:
   - Read back the item, the amount, and the refund method.
   - Accept "yes" only on Flux's final `EndOfTurn` signal, never the early `EagerEndOfTurn`.
   - The read-back must have played in full.
   - Short sounds like "mm-hmm" never count.
   - Anything unclear means ask again, or use the on-screen button.
4. Tune interruption so playback stops within 300 ms. Check LiveKit's `min_interruption_duration` default, which may be 0.5 s. Try LiveKit's adaptive interruption handling.
5. Set Sonnet 5's effort level lower for voice turns.
6. Build the browser UI: mic permission, start/stop/mute, captions, speaking and listening state, and a text fallback. When the connection drops, reconnect and read the latest return status.
7. Build a transcript simulator that runs the same graph and UI states without voice keys.
8. Enforce per-user caps on voice minutes per day, concurrent sessions, and LLM tokens per conversation.
9. **Voice monitoring:**
   - Turn on LiveKit tracing with `set_tracer_provider(..., allow_pii=False)`.
   - LiveKit's OTel export covers traces only. Its per-stage timings are SDK metric events (`EOUMetrics.end_of_utterance_delay`, `LLMMetrics.ttft`, `TTSMetrics.ttfb`). Write a small handler that turns them into OTel metrics.
   - Measure interruption stop time ourselves.
   - Check these metric names against the pinned `livekit-agents` version. They were read from LiveKit's main branch, not the 1.8.3 release.
   - Build the "Voice" dashboard, with an alert for voice p95 over 2 s.
10. **Voice evals (layer 5):**
    - Record at least 30 audio clips in `evals/audio/`: accents, noise, pauses, barge-in, and "mm-hmm".
    - Build a replay runner that sends them through the real pipeline.
    - The transcript simulator also runs the text versions in CI.

**Exit criteria:**

- With keys, a spoken return goes through the same quote and confirm route as text.
- Background speech and "mm-hmm" never confirm.
- Interrupting the read-back does not confirm.
- A reconnect shows the current status.
- Latency is measured from the real end of speech, and the numbers are reported honestly.
- The audio replay shows **0 false confirmations**. Per-stage latency is visible on the Voice dashboard.

### Step 7 — Milestone 5: hardening (~7d)

**Why:** It turns a demo into something safe to pilot.

1. **Security:**
   - CSRF protection on Route Handlers; the webhook route is exempt.
   - A per-request CSP nonce.
   - Rate limits and CAPTCHA on sign-in, checkout, and `/v1/voice/token`.
   - Secret scanning in CI.
2. **Privacy:**
   - Hide personal data in logs and LangSmith traces.
   - Audio recording off by default.
   - A delete-my-data job covering PostgreSQL, transcripts, traces, and Pinecone.
3. **Accessibility:** run a WCAG 2.2 AA check with axe and Playwright. Check keyboard use, focus, target size, and screen-reader announcements for streaming chat and status updates.
4. **Evals, full strength:**
   - Grow to at least 200 scenarios and 100 policy questions.
   - Add layer 4 (conversation quality: Ragas agent metrics plus the judge).
   - Add layer 6 (the red-team attack set).
   - Check the judge against 50 human-labeled examples, and record how often they agree.
   - Turn on layer 7 (live evals on sampled, personal-data-free traces, plus thumbs up and down).
   - Build the review queue that turns failed conversations into new scenarios.
   - Record the full baseline: intent accuracy, item selection, citation accuracy, Ragas scores, false confirmations, handoff rate, and cost per resolved task.
5. **Monitoring, full strength:**
   - Set the service objectives and warn alerts from section A.
   - Tune tail sampling.
   - Set up ADOT to CloudWatch/X-Ray in the Terraform proposal.
   - Write an on-call runbook: for each alert, what it means and what to check first.
   - Run the load tests with monitoring on, so the dashboards show the bottleneck.
6. **Jev experiment** (optional, behind a flag): compare it with the baseline router on the same scenarios.
7. **Load tests** with k6 at the pilot assumptions (500 requests/s, 20 checkouts/s, 50 voice sessions). Report hardware, p50/p95/p99, errors, and the bottleneck.
8. **Admin:** a basic catalog and policy-version editor for the staff role.
9. **Cloud:** a Terraform proposal for ECS Fargate, ALB, RDS, and the rest, targeting **Taro's personal AWS account only**.
   - The provider uses `profile = "personal"` and `allowed_account_ids = ["<personal account ID>"]`.
   - A `make aws-whoami` check runs `aws sts get-caller-identity` and stops if the account ID doesn't match.
   - Do not deploy until Taro approves it.

**Exit criteria:**

- The acceptance suite is green.
- The eval baseline and load results are written up, including any targets that were missed.
- The red-team set shows **0 unsafe actions**.
- Every alert has been fired once on purpose in a drill, and the on-call runbook covers it.
- A deployment runbook and release checklist exist.

### Step 8 — Human release review (~1d)

**Why:** Only a person can approve real business rules and live keys.

1. Review the return rules, tax registrations, payment capture timing, fraud and manual-review rules, privacy and retention rules, and support escalation.
2. Run `/cold-review` on the final branch, and fix anything High or worse.
3. Write the status report: what is connected versus simulated, the test counts, and the known gaps.

**Exit criteria:** The report and checklist are approved.

---

## File-by-File Summary

| Path | Change | Details |
|---|---|---|
| `CLAUDE.md` | New | Points Claude Code at `docs/handoff/AGENTS.md` |
| `docs/handoff/*.md` | Edit | Step 1 fixes: confirmation, Pinecone, Stripe Tax, states, treg removed |
| `apps/web/` | New | Next.js storefront, account pages, assistant and voice panel |
| `services/commerce/` | New | FastAPI catalog, cart, checkout, orders, returns, webhooks, dev endpoints |
| `services/commerce/search/` | New | OpenSearch plus Pinecone hybrid search, and reindex command |
| `services/assistant/` | New | LangGraph graph, typed tools, SSE streaming, Pinecone policy retrieval |
| `services/voice/` | New | LiveKit worker, Flux, Cartesia, voice-confirmation state |
| `workers/workflows/` | New | Temporal checkout workflow, reconciliation schedule, return workflow, and refund activity |
| `apps/web/proxy.ts` | New | Sign-in wall: sends signed-out visitors to `/sign-in?returnTo=…`, and allows only the public pages |
| `apps/web/app/(auth)/` | New | Sign-in, create account, forgot password, and sign-out pages |
| `services/commerce/auth/` | New | Development sign-in adapter, Cognito OIDC adapter, session checks on every route |
| `workers/outbox/` | New | Polling publisher that feeds OpenSearch and Pinecone |
| `packages/contracts/` | New | Generated TypeScript client and shared schemas |
| `evals/` | New | Scenarios, `policy_qa.jsonl`, `search_queries.jsonl`, audio, red-team set, code graders, Ragas runner, audio replay runner |
| `.github/workflows/evals.yml` | New | Fast eval gate on pull requests; full suite plus Ragas nightly |
| `infra/otel/collector.yaml` | New | OTel Collector: receive, remove personal data, tail-sample, export |
| `infra/grafana/` | New | The 4 dashboards and alert rules, stored as code |
| `apps/web/instrumentation.ts` | New | Next.js OTel setup plus browser Web Vitals |
| `packages/telemetry/` | New | Shared Python OTel setup and span labels, used by every service and worker |
| `docs/runbook-oncall.md` | New | For each alert: what it means and what to check first |
| `tests/` | New | pytest (unit and integration), Playwright (browser), k6 (load) |
| `infra/` | New | Docker Compose, Terraform proposal |
| `.env.example` | New | Every variable, with fake placeholder values |
| `.github/workflows/ci.yml` | New | Lint, type checks, tests, secret scanning |

---

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Developers' local data collides in the shared Pinecone index | Med | Med | Namespace per environment and developer (`products-dev-taro`); `make reindex` rebuilds from PostgreSQL. |
| Every developer and CI run needs a Pinecone key, so there is no offline work | High | Low | Accepted by decision. Pure unit tests use a small stand-in at the adapter boundary (approved 2026-09-25). Integration tests and evals use a real `ci` namespace. |
| Cloud resources get created in a RealPage AWS account by mistake | Low | High | Terraform `allowed_account_ids` pinned to the personal account; the `make aws-whoami` guard; no RealPage AWS profile on this machine (checked 2026-09-25). |
| A surprise AWS bill on the personal account | Med | Med | Billing alarm before any resource; `make destroy` documented; the cheapest sizes in the Terraform proposal. |
| Someone bypasses the `proxy.ts` sign-in wall and reaches data | Med | High | Every API route checks the session itself; a test calls the APIs directly with no session and expects 401. |
| `returnTo` is abused to send users to a fake site after sign-in | Med | Med | Accept only paths on our own site; a test tries an outside URL. |
| Sign-in wall hurts first impressions (visitors can't look around first) | Med | Low | Accepted by decision. The sign-in screen explains what the store is, and creating an account takes one short form. |
| Temporal checkout adds moving parts before the first sale | Med | Med | Keep the workflow small (one reservation, one session, one order); use the time-skipping test environment; the Temporal dev server is already in Compose. |
| The unit-test stand-in slips into the running app | Low | Med | It lives only in `tests/unit/`. No app setting can select it. A CI check fails if app code imports it. |
| Stripe Tax shows $0, so tax tests pass for the wrong reason | Med | Med | Add a test state registration in the pre-flight; a test asserts tax is greater than 0. |
| Stripe Tax handles partial refunds differently than expected | Med | High | Step 4.6 checks this before refunds are built on it. |
| The model finds a way to confirm a return | Low | High | No confirm tool exists. The token is never in model context or checkpoints, and the browser fetches it separately. The API checks the approval record itself, and a test proves all of this. |
| Stripe payment succeeds after the stock hold expires | Med | High | Session `expires_at` matches the reservation. If a late payment still arrives, re-reserve the stock or refund automatically and flag it for staff. |
| A refund retry after 24 hours makes a second refund | Low | High | Check Stripe before every retry, plus a deterministic operation ID. |
| Voice "mm-hmm" or background speech confirms a refund | Med | High | Final end-of-turn only, full read-back, short sounds rejected, button fallback. |
| Voice latency misses p50 under 1 s | High | Med | Lower model effort, same-region providers, streaming, honest reporting. The targets are goals, not promises. |
| Someone runs up the AI bill ("denial of wallet") | Med | Med | Per-user minute, session, and token caps; rate limits on the token endpoint. |
| LangChain approval middleware re-runs rejected tool calls (open issue #40492) | Med | High | Don't use that middleware for confirmation; use `interrupt()` plus the app-controlled confirm endpoint. |
| Personal data leaks into traces, LangSmith, or the Ragas judge | Med | High | The collector removes personal data; customer IDs are hashed; evals use synthetic customers; a CI test scans exported spans for emails and card-like numbers. |
| The AI judge (in Ragas or the LLM judge) scores unreliably | Med | Med | Safety gates use code checks only. Judge scores become gates only after comparison with 50 human labels. |
| Evals get "taught to the test" | Med | Med | A held-out test split that is never used for tuning; new scenarios keep arriving from live failures. |
| Eval runs cost too much (judge tokens, voice minutes) | Med | Low | Only a fast set runs on pull requests; the full set runs nightly; voice replay runs weekly; each run logs its cost. |
| Trace storage costs grow | Low | Low | Tail sampling keeps 100% of errors and money traces but only 10% of normal ones. |
| Ragas or OTel GenAI naming changes between versions | High | Low | Already happening: Ragas 0.4 renamed metrics and deprecated `evaluate()`, and the GenAI names are still in "Development" status. Pin versions in lockfiles; keep all Ragas calls in `evals/ragas_runner.py`. |

---

## Rollback Plan

Nothing is deployed live, so rollback is local.

1. **Code:** each milestone is its own commit or branch. Use `git revert` to undo the milestone commit.
2. **Database:** run `alembic downgrade -1` for each migration in that milestone, newest first.
3. **Search indexes:** run `make reindex` to rebuild OpenSearch and Pinecone from PostgreSQL. To reset, delete the environment's Pinecone namespace.
4. **Stripe:** test-mode data only. Clear it from the Stripe test dashboard if needed.
5. **Feature flags:** turn off the Jev and voice flags to fall back to text chat.

---

## Success Criteria (Definition of Done)

- [ ] The README demo works from a fresh clone: sign in, buy, then return by text, then by voice (or simulator), then see the refund status.
- [ ] Every page except the public list sends a signed-out visitor to sign-in, and every private API returns 401 with no session (tested).
- [ ] Checkout runs as a Temporal workflow and survives a worker crash (tested).
- [ ] Every acceptance test in `ACCEPTANCE_AND_EVALS.md` passes, and the counts are shown.
- [ ] Zero unauthorized or duplicate refunds in the adversarial tests.
- [ ] Tests prove the model has no confirm tool, and no confirmation token appears in model messages, tool results, or checkpoints.
- [ ] Pinecone is the only vector store (`grep` finds no other).
- [ ] No treg code or config exists.
- [ ] The report states clearly which providers were really connected and which were simulated.
- [ ] Load, latency, and eval results are reported with real numbers.
- [ ] Every customer action can be followed as one OTel trace, from the browser to Stripe or the LLM and back.
- [ ] No personal data appears in exported traces or logs (tested).
- [ ] The eval CI gate blocks unsafe changes; Ragas, safety, search, voice, and red-team baselines are recorded.
- [ ] Live evals and the four dashboards are running; every alert has been drilled once.
- [ ] The WCAG 2.2 AA check passes, or any gaps are listed.
- [ ] The human release review is signed off.
