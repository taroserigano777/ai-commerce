# Implementation plan and milestone prompts

Use the master prompt first. These shorter follow-ups can be pasted into the same coding session, one at a time. The model should show a diff/commit, commands run, and observed results at each gate.

## Milestone 1: Runnable shopping foundation

**Build:** Monorepo, lockfiles, local setup, migrations, seed products and policy, responsive Next.js pages, FastAPI catalog/cart/orders modules, PostgreSQL, product search indexing, local auth adapter and production OIDC interface, generated API client. Provide `.env.example` and Compose. Implement a working browse → product → cart path.

**Gate:** Fresh checkout starts with documented commands; seed migration succeeds; browser journey works; search and price/stock refresh are correct; a second user cannot access the first user's cart.

**Follow-up prompt:** “Implement Milestone 1 from IMPLEMENTATION_PLAN.md. Run it from a fresh local setup, fix failures, and report the URL, commands, evidence, and commit. Continue to Milestone 2 if the gate passes.”

## Milestone 2: Checkout and orders

**Build:** Server-side price/tax/shipping quote (sandbox assumptions documented), atomic stock reservation, Stripe test-mode checkout, verified webhooks, idempotent attempts, payment/order state, order page, reconciliation, outbox. If Stripe keys are unavailable, finish the code and exercise a provider simulator in tests; clearly mark connected payment unverified.

**Gate:** Test-mode success/failure, duplicate submit, changed price, expired stock reservation, duplicate/out-of-order webhook and wrong-user access behave correctly. An order is PAID only after verified provider evidence.

**Follow-up prompt:** “Implement Milestone 2. Focus on checkout correctness and idempotency. Use Stripe test mode; if keys are unavailable, use a labeled simulator and report the precise untested connected path. Show targeted tests.”

## Milestone 3: Returns and text assistant

**Build:** Versioned policy and eligibility rules, return quote, explicit quote-bound confirmation, durable return operation via Temporal, status tracking, refund simulation/Stripe test refund when allowed, staff exception queue, LangGraph text chat, approved policy retrieval with citations, typed scoped commerce tools.

**Gate:** One sandbox order can be returned once; altered/expired quotes and repeat confirmation cannot create extra returns/refunds. Chat clarifies an ambiguous item. Private order data stays private. The FAQ answer cites an approved policy version.

**Follow-up prompt:** “Implement Milestone 3 with a complete text-guided return. The backend decides eligibility and amounts. Add tests for ownership, repeat delivery, quote tampering, policy expiry, and refund status. Demonstrate the end-to-end slice.”

## Milestone 4: Voice

**Build:** LiveKit room token endpoint, React mic UI, Python worker, Deepgram Flux, Cartesia, shared LangGraph/tool code, captions, interruption, confirmation card, reconnection and text fallback. Provide a transcript simulator for development without credentials.

**Gate:** With credentials, spoken return uses the same quote/confirmation route as text. Without credentials, the simulator exercises the same graph and UI states. Interrupted speech does not commit a return; a disconnected client resumes status. The connected path must be reported as unverified if not actually run.

**Follow-up prompt:** “Implement Milestone 4. Wire the real voice providers behind environment configuration, keep a no-credential transcript simulator, and test interruption, ambiguity, reconnection, and exact-quote confirmation.”

## Milestone 5: Hardening and optional experiments

**Build:** OpenTelemetry/LangSmith tracing with PII controls, load tests, accessibility, failure drills, staff handoff, cloud infrastructure proposal/config, Jev intent-routing and treg approved-feed experiments under flags. No need to enable the experiments by default.

**Gate:** Compare Jev with baseline on the same labeled transcripts; measure latency, accuracy and uncertain routing. Test one treg-supported licensed endpoint and direct-provider alternative, including cost and failure modes. Load-test against the stated pilot assumptions and report achieved values honestly.

**Follow-up prompt:** “Implement Milestone 5. Run the acceptance suite and targeted load/eval tests. Keep Jev/treg optional. Produce a deployment runbook, measured results, unresolved risks, and a release checklist for human review.”

## Definition of a useful report

Each milestone response should state: implemented files and user-visible behavior; exact setup/run/test commands; passing/failing counts; screenshots or URLs if available; external services actually connected; known gaps; next step. Do not summarize a simulated workflow as a live provider test.
