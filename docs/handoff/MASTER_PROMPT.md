# Master prompt for GPT-6 Sol or Claude Opus 5.5

You are the senior engineer responsible for implementing the attached AI commerce project. Read README.md, PRODUCT_REQUIREMENTS.md, TECHNICAL_SPEC.md, IMPLEMENTATION_PLAN.md, ACCEPTANCE_AND_EVALS.md, AGENTS.md, and ecommerce-architecture-plan.md first. Treat the handoff files as the requirements; the architecture plan supplies rationale and source links. If they differ, follow the more specific requirements in this package and document the discrepancy.

## Goal

Build an original, polished e-commerce web app using React + TypeScript + Next.js, a Python FastAPI backend, PostgreSQL, product search, Stripe test-mode payments, and a text/voice assistant that handles general questions, order lookup, and guided returns. Use LangGraph and LangChain for assistant orchestration. Use LiveKit + Deepgram Flux + Cartesia for connected voice. Use Temporal for durable return/refund business workflows. Jev and treg are **optional experiments**, not dependencies for the first working purchase/return slice.

The first complete journey is: seed product → browse/search → cart → server-priced checkout → Stripe test payment → order page → ask assistant to return an item → backend checks eligibility and creates a quote → explicit confirmation → durable return record → label/mock fulfillment event → refund attempt and accurate status. General information questions must answer from approved policy text with a visible citation. Voice and text share the same business tools. The UI shows listening/speaking state, transcript/captions, item cards, and a text fallback.

## How to work

1. Inspect the repository and environment. If it is empty, scaffold it. If code exists, preserve useful work and explain any changes to the proposed structure.
2. Write a short execution plan with milestones and dependencies, then **implement** the first milestone. Do not stop after a design or a scaffold. If the environment allows, continue through subsequent milestones in order.
3. Use a modular commerce backend first; deploy the web app, commerce API, AI API, and voice workers separately when cloud deployment is in scope. Keep clear module boundaries without inventing dozens of microservices.
4. Commit at meaningful milestones if a Git repo is available. After each milestone run its targeted checks and report evidence. Fix observed failures before moving on.
5. Resolve ordinary implementation choices yourself. Ask only if a decision changes business policy or needs unavailable secrets/accounts. Use documented development defaults for missing credentials. Never claim an external provider integration works if it was not tested with valid credentials.
6. Produce runnable code, migrations, seed data, Docker Compose for local services, a reproducible README, `.env.example`, API docs/OpenAPI, and a status report. Do not expose secrets.
7. Keep a clear boundary between model suggestions and commerce actions. An LLM or Jev result cannot authorize a refund, invent a refund amount, or bypass order ownership, quote confirmation, or backend policy. Use idempotency keys and reconciliation for payments and refunds.
8. Keep Redis cache-only, Pinecone as the production policy vector store, and MCP optional. Use AWS ECS Fargate and an Application Load Balancer for the proposed deployment; no Kubernetes.

## Acceptance for the first delivery

Provide a URL or local command that opens the storefront. A developer must be able to run tests and the seeded purchase/return demo from the README. Demonstrate or document the exact point where provider credentials are needed. Include automated coverage of unauthorized order access, tampered totals, duplicate checkout/return requests, stale quotes, failed/refunded payments, and interrupted or uncertain voice confirmation. Show actual test output and distinguish simulated providers from connected ones.

Start now by reading the files and implementing Milestone 1. If you have the tools and time, continue to the next milestone after its exit gate passes.
