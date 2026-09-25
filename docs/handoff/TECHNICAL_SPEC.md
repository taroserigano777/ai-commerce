# Technical specification

## Stack and service boundaries

| Concern | Choice | Ownership |
|---|---|---|
| Web | React, TypeScript, Next.js App Router, Tailwind, shadcn/ui, TanStack Query for interactive server state | Storefront and voice UI |
| Commerce | Python, FastAPI, Pydantic, SQLAlchemy, Alembic | Catalog, pricing, cart, stock, checkout, orders, returns |
| Data | PostgreSQL; Redis only as cache | Source of truth and expendable read cache |
| Product discovery | OpenSearch with keyword, filters, optional semantic ranking | Rebuildable catalog index |
| Assistant | FastAPI, LangGraph, LangChain, PostgreSQL checkpointer | Text and voice conversational state, narrow tools |
| Policy retrieval | Pinecone with version/effective-date metadata; local deterministic substitute in development | Approved policy text and citations |
| Voice | LiveKit Cloud + Python LiveKit Agents + Deepgram Flux + Cartesia | WebRTC session, transcript, speech, interruption |
| Business process | Temporal Python workers | Checkout/return wait states, label and refund retries |
| Payment | Stripe test mode during development | Payment intent/session and provider refund |
| Events | PostgreSQL transactional outbox, EventBridge/SQS in AWS | Search projection and notifications |
| Identity | Cognito/OIDC in cloud; development identity adapter | Authenticated subject and server-side roles |
| Runtime | Docker Compose locally; AWS ECS Fargate + ALB, S3/CloudFront, RDS/Aurora for cloud | Independent web/API/voice/worker scaling |
| Telemetry | OpenTelemetry, CloudWatch, LangSmith | Request, workflow, and assistant diagnostics |

Use a monorepo such as `apps/web`, `services/commerce`, `services/assistant`, `services/voice`, `workers/workflows`, `packages/contracts`, `infra`, `tests`. A different clean layout is acceptable if justified. Use current compatible stable releases and pin them in lockfiles; do not hardcode version claims from the research date.

## Request paths

Shopping: browser → Next.js → commerce API → PostgreSQL. Product search reads OpenSearch, then checkout rechecks authoritative PostgreSQL state. AI text: browser → assistant API (stream response via SSE) → LangGraph → authorized typed commerce tool/policy retrieval. Voice: browser ↔ LiveKit ↔ Python voice worker → shared LangGraph package → the same tools. LiveKit media does not travel through the HTTP load balancer. Temporal workers poll their task queues; they do not expose a public HTTP endpoint.

## Suggested tables and invariants

- `users`, `products`, `variants`, `categories`, `prices`, `stock_locations`, `inventory_balances`, `inventory_reservations`.
- `carts`, `cart_items`, `checkout_attempts`, `orders`, `order_lines`, `payment_attempts`, `shipments`.
- `return_quotes`, `return_requests`, `return_items`, `refund_attempts`, `support_cases`, `policy_versions`.
- `conversations`, `messages`, `graph_checkpoints`, `tool_audit`, `outbox_events`, `provider_webhook_events`.

Use integer minor currency units and ISO currency codes. Store purchase-time price, tax, merchant, and applicable policy reference with each order line. Constrain return quantity to purchased minus already returned. Add unique business-operation keys to checkout, return, refund and webhook records. Keep one current inventory balance per variant/location and reserve atomically. Do not treat search/cache as authoritative.

## API contract sketch

All private routes require a verified identity; enforce ownership on every lookup and mutation. IDs are opaque. Publish actual OpenAPI schemas and generate the TypeScript client.

| Method and path | Input | Output/behavior |
|---|---|---|
| `GET /v1/products` | query, filters, pagination | Product summaries and facets; search-backed |
| `GET /v1/products/{id}` | product ID | Variants and descriptive details; fresh price/stock fields |
| `GET /v1/cart` / `PUT /v1/cart/items` | variant, quantity | Durable cart; server-side current estimate |
| `POST /v1/checkout/quote` | cart/version, address or region | Server totals, tax/shipping treatment, stock validation, quote expiry |
| `POST /v1/checkout/start` | quote ID, idempotency key | Checkout attempt and Stripe test-mode client/session reference |
| `GET /v1/orders` / `GET /v1/orders/{id}` | authenticated customer | Own orders only |
| `POST /v1/returns/quote` | order line, quantity, reason, condition | Structured eligibility result and exact quote |
| `POST /v1/returns/confirm` | quote ID, bound confirmation token, idempotency key | One return ID and workflow status |
| `GET /v1/returns/{id}` | return ID | Authorized return plus separate refund state |
| `POST /v1/assistant/messages` | conversation, text | SSE text/event stream and structured UI cards |
| `POST /v1/voice/token` | authenticated request | Short-lived, room-scoped LiveKit token |
| `POST /v1/webhooks/stripe` | verified provider payload | Persist/queue and deduplicate event; acknowledge promptly |

Return quotes should include `quote_id`, `order_line_id`, `quantity`, `refund_amount_minor`, `currency`, `deductions`, `method_options`, `inspection_required`, `policy_version`, `expires_at`, `status`, and a confirmation binding. Expired, changed, or already consumed quotes require a new quote. Model-facing tools expose `list_my_orders`, `get_order_status`, `get_return_options`, `confirm_return`, `search_products`, `get_product_details`, and `create_support_case` through typed wrappers; never a free-form SQL/refund tool.

## State and retries

Checkout states: CREATED → STOCK_RESERVED → PAYMENT_PENDING → PAID → FULFILLING, with explicit FAILED/EXPIRED/CANCELED branches. Decide payment capture timing and record it. Return states: REQUESTED/AUTHORIZED → IN_TRANSIT → RECEIVED → INSPECTED → CLOSED, or REVIEW/REJECTED/CANCELED. Refund states: NOT_REQUESTED → REQUESTED → PENDING → SUCCEEDED or FAILED. Status labels must reflect observed provider/business state.

Persist business change plus outbox event in the same PostgreSQL transaction. Repeated webhooks and workflow deliveries are expected. Every external action gets a deterministic operation ID. After an ambiguous timeout, query Stripe/shipping state before retrying. Temporal coordinates waits and retries but does not itself guarantee exactly-once external effects. A voice disconnect or interrupted LangGraph node must not repeat a committed return.

## Voice and assistant details

Issue short-lived room tokens from the authenticated server. Deepgram Flux supplies transcripts and turn completion; LiveKit supports media and interruption; Cartesia synthesizes the approved response. Share LangGraph definitions/tools between text and voice. Persist a conversation ID and serialize simultaneous text/voice turns. Preemptive work may perform read-only retrieval; never trigger side effects before intent and confirmation are settled. On barge-in, stop output and reconcile any in-flight operation before speaking again.

Separate response types: `assistant_text`, `policy_citation`, `product_card`, `order_card`, `return_quote_card`, `operation_status`, `handoff`. The client renders structured amounts and statuses from backend data. For a public question, retrieve versioned policy content and cite it. For a live fact, use an authenticated API. If evidence conflicts or the user asks for an exception, escalate.

Jev can be tried behind a feature flag for bounded intent choices and support triage after a baseline exists. Its probabilities are signals, not authorization. treg can be tried behind an allowlisted integration adapter for an approved external supplier feed. Neither is on the critical purchase/refund path, nor is a general treg token given to the LLM. Compare latency, accuracy, spend, and reliability before adopting either.

## Local and cloud operations

Local Compose should provide PostgreSQL, Redis, OpenSearch, and a Temporal dev service if needed; web/API/worker commands must be documented. Use deterministic fixtures for policy retrieval and a transcript simulator when Pinecone or voice accounts are unavailable. Stripe test mode is required for a connected checkout demonstration; a clearly labeled payment simulator may exercise early development/tests only. Seed products, stock, policies and a test customer.

Cloud proposal: CloudFront/WAF → Next.js ECS and ALB → FastAPI commerce/assistant ECS; separate voice and Temporal workers; Aurora/RDS PostgreSQL, managed Redis, managed OpenSearch, Pinecone, LiveKit, Stripe, S3, EventBridge/SQS. Keep credentials in a secrets manager. Configure backups, health checks, draining, provider quotas, and separate scaling for voice and HTTP workloads. Do not deploy without a reviewed environment and real account ownership.
