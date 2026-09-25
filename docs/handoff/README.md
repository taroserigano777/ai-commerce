# AI commerce build handoff

This package is a build-ready brief for a coding agent such as GPT-6 Sol or Claude Opus 5.5. It describes an original retail application with a familiar large-store shopping experience. It is **not** a Walmart clone, a production certification, or a request to scrape/copy another retailer's designs or data.

## Start here

1. Put this folder in the root of a new Git repository, or attach the zip to a coding session with repository access.
2. Give the model `MASTER_PROMPT.md`. Tell it to read every file in this folder before editing.
3. If working locally, give it an isolated development environment, Docker, and permission to run tests. Do not paste real keys into the prompt. Configure secrets through environment variables or a secret manager.
4. Ask it to implement **Milestone 1** end to end and report exact commands, test results, and remaining blockers. Continue through the gates in `IMPLEMENTATION_PLAN.md` if time and budget allow.
5. Review payment, refund, identity, tax, fulfillment, and policy settings before live use. The app should default to development/sandbox providers.

## Files

| File | Purpose |
|---|---|
| `MASTER_PROMPT.md` | Copy-paste task for either coding model |
| `PRODUCT_REQUIREMENTS.md` | Users, journeys, scope, behavior, UX |
| `TECHNICAL_SPEC.md` | Stack, boundaries, APIs, data, reliability, voice |
| `IMPLEMENTATION_PLAN.md` | Ordered milestones and proof for each |
| `ACCEPTANCE_AND_EVALS.md` | Test cases, AI evaluation, performance targets |
| `AGENTS.md` | Persistent coding-agent instructions for this project |
| `ecommerce-architecture-plan.md` | Prior researched architecture and primary-source links |

## Working assumptions to confirm

US and English at launch; first-party products; one warehouse/fulfillment adapter; USD; web storefront; no mobile app; no marketplace or grocery substitutions. Initial load-test assumptions are 100k variants, 100k accounts, 500 peak dynamic requests/s, 20 checkout attempts/s, and 50 voice sessions. These are sizing hypotheses, not measured capacity.

The first deliverable is a runnable vertical slice: browse a seeded product, add it to cart, complete a Stripe test-mode purchase, see the order, initiate an eligible return through text or voice, confirm the exact quote, and see the return/refund status. Live voice requires provider credentials; a local text and transcript simulator must be usable without them.

## Minimum accounts and secrets for connected demonstrations

- Stripe **test-mode** account and webhook signing secret.
- LiveKit Cloud project, Deepgram, Cartesia, and an LLM provider account for actual spoken conversations.
- Pinecone index and embeddings provider for hosted policy retrieval.
- AWS account only for cloud deployment; local Docker development must not require it.
- Optional TypeSafe Jev access and optional treg account for later experiments. Neither blocks the main path.

Document each variable in `.env.example` with a fake placeholder; never include live credentials in Git or prompts.
