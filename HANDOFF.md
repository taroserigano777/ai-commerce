# Handoff — AI Commerce Build

**Written:** 2026-09-25, on Taro's Mac, at the end of a planning session.

**Read this first** if you are continuing on another machine (for example, the Windows laptop). Claude's memory notes from the Mac do **not** move with the files, so every decision is written here.

---

## Where things stand

| Item | Status |
|---|---|
| Review of the original handoff package (8 docs) | ✅ Done |
| Research: fact check of all 22 sources, best practices, OTel and Ragas facts | ✅ Done. Results are built into the plan. |
| Build plan | ✅ Done: `docs/PLAN_ai-commerce-build.md` |
| Step 0: project folder, docs unzipped, `CLAUDE.md`, `.gitignore`, `.gitattributes` | ✅ Done |
| First Git commit, pushed to GitHub | ✅ Done: **private** repo `https://github.com/taroserigano777/ai-commerce`, branch `main` |
| Step 1: fix the handoff docs | ⬜ Next |
| Steps 2–8: build | ⬜ Not started |
| Application code | ⬜ None yet |

---

## What's in this folder

```
ai-commerce/
├── HANDOFF.md                     ← this file
├── CLAUDE.md                      ← one line: @docs/handoff/AGENTS.md
├── .gitignore                     ← ignores .env, node_modules, .venv, etc.
├── .gitattributes                 ← forces Linux (LF) line endings
└── docs/
    ├── PLAN_ai-commerce-build.md  ← THE plan (steps 0–8, about 40 days)
    └── handoff/                   ← original 8 docs, not yet edited
        ├── README.md
        ├── MASTER_PROMPT.md
        ├── AGENTS.md
        ├── PRODUCT_REQUIREMENTS.md
        ├── TECHNICAL_SPEC.md
        ├── IMPLEMENTATION_PLAN.md
        ├── ACCEPTANCE_AND_EVALS.md
        └── ecommerce-architecture-plan.md
```

**Important:** where the plan and `docs/handoff/` disagree, **the plan wins**. Step 1 exists to update the handoff docs to match it.

---

## All decisions made (these override the original handoff)

| # | Topic | Decision |
|---|---|---|
| 1 | Vector database | **Pinecone only, no exceptions.** It holds both policy/help text and product "meaning" vectors. No pgvector, and no OpenSearch vectors. |
| 2 | Pinecone in development | A **real Pinecone account**, even locally. Each developer and environment gets its own namespace. |
| 3 | Pinecone in tests | Pure unit tests only may use a small stand-in, kept in `tests/unit/`. The app, integration tests, evals, and the demo always use real Pinecone. |
| 4 | Product search | OpenSearch handles keywords and filters. Pinecone handles meaning. The results are merged with reciprocal rank fusion (a standard way to combine two ranked lists). Price and stock always come from PostgreSQL. |
| 5 | Sales tax | **Stripe Tax**, inside Stripe Checkout Sessions. Our server does not calculate tax. |
| 6 | treg | **Dropped entirely.** Its data is other retailers' listings, including Walmart's and Amazon's. |
| 7 | Monitoring | **OpenTelemetry everywhere**, built from Milestone 1. |
| 8 | Evals | A strong 7-layer eval system that includes **Ragas**. Safety checks use plain code and block merges in CI. |
| 9 | Sign-in | **Sign-in comes first.** The sign-in screen is the first thing built. Every visitor must sign in before using any part of the site. |
| 10 | Guests | **No guest access at all**: no guest browsing, cart, or checkout. |
| 11 | Temporal | **Used in Milestone 2** for the checkout workflow, and in Milestone 3 for returns. |
| 12 | AWS | **Taro's personal AWS account only, never RealPage's.** Use a CLI profile named `personal`, and pin Terraform with `allowed_account_ids`. |

**Defaults Taro accepted** (from the plan): Stripe Checkout Sessions with cards only; refunds go only to the original card; WCAG 2.2 AA; Jev as an optional flagged experiment; the sandbox return policy in Step 1.

---

## Open items

1. **Git email for commits.** This repo uses the GitHub no-reply address `275150514+taroserigano777@users.noreply.github.com`, set for this repo only, so the work email stays out of a personal repo. **Set it again after cloning on Windows**, because repo settings don't travel with `git clone`:

   ```bash
   git config user.email "275150514+taroserigano777@users.noreply.github.com"
   git config user.name  "taroserigano777"
   ```

   To use a different email, just set a different value.

2. **Personal AWS account ID** (12 digits). It is needed for Terraform's `allowed_account_ids`, but only in Step 7.

3. **Accounts and keys** to get before the steps that need them:
   - Stripe test mode, plus the Stripe CLI, and Stripe Tax with a test state registration.
   - Pinecone.
   - An LLM key (start with Claude Sonnet 5).
   - LiveKit, Deepgram, and Cartesia (Step 6).
   - Personal AWS (Step 7).

---

## Moving to the Windows laptop

The repo is on GitHub (private, personal account). On Windows, inside WSL:

```bash
gh auth login        # sign in as taroserigano777
gh repo clone taroserigano777/ai-commerce ~/ai-commerce
cd ~/ai-commerce
git config user.email "275150514+taroserigano777@users.noreply.github.com"
git config user.name  "taroserigano777"
```

### Windows setup (recommended)

| What | Why |
|---|---|
| **WSL2 with Ubuntu** | The stack (Docker, Python, Node, Temporal, shell scripts) runs much more smoothly on Linux. |
| Keep the repo **inside WSL** (for example `~/ai-commerce`), **not** under `C:\` or `/mnt/c/...` | Files under `/mnt/c` are slow for Docker and Node, and file watching is unreliable there. |
| **Docker Desktop** with the WSL2 backend turned on | Runs PostgreSQL, Redis, OpenSearch, the Temporal dev server, and the OTel/Grafana stack. |
| Give Docker at least 8 GB of RAM (16 GB is better) | OpenSearch alone needs about 2 GB or more. |
| Inside WSL, install Git, Node LTS with pnpm, Python 3.12 or newer with `uv`, the Stripe CLI, and later the AWS CLI | These are the build tools. |
| Line endings: the repo already has a `.gitattributes` with `* text=auto eol=lf`. Also run `git config --global core.autocrlf input` in WSL. | Stops Windows line endings from breaking shell scripts and Docker builds. |
| Install Claude Code inside WSL, and open the repo from there (or through VS Code's WSL extension) | Claude then sees the same Linux paths as the build. |

---

## Next steps, in order

1. **Clone on Windows** and set the repo Git email (see "Moving to the Windows laptop").
2. **Step 1.** Edit `docs/handoff/*.md` to match the plan. The plan's Step 1 table lists every edit. The exit check is:

   ```bash
   grep -riE "treg|pgvector|PostgreSQL vector search|deterministic substitute" docs/handoff
   ```

   It must find nothing. Commit.
3. **Step 2 (Milestone 1).** Build sign-in first, then the shopping foundation, monitoring, and search evals. Follow the plan.

---

## Prompt to paste into Claude Code on Windows

```text
Read HANDOFF.md, then docs/PLAN_ai-commerce-build.md, then every file in docs/handoff/.
HANDOFF.md and the plan override docs/handoff where they differ.
Save the "All decisions made" table from HANDOFF.md to your memory for this project.
Step 0 is done and pushed. Do Step 1 (update docs/handoff to match the plan).
Stop after Step 1 and show me the diff before committing.
```

---

## Useful facts verified on 2026-09-25

These are already in the plan. Re-check the versions when you pin lockfiles.

- **Newest versions:** Next.js 16.3.6 · livekit-agents 1.8.3 · langgraph 1.2.12 · langchain 1.4.2 · temporalio 1.33.0 · fastapi 0.141.1 · ragas 0.4.3.
- **Ragas 0.4 changed its API.** Use `ragas.metrics.collections`, `@experiment`, and `ascore()`. **Don't use `evaluate()`**: it is deprecated and breaks with the new metrics. Claude can be the judge through `llm_factory(..., provider="anthropic")`, with the `anthropic` package installed separately.
- **Stripe:**
  - Checkout Sessions is Stripe's recommended option.
  - Refund states include `requires_action` and `canceled`, and a succeeded refund can still fail later.
  - Stripe forgets idempotency keys after 24 hours, so always check Stripe before retrying.
  - Test-mode tax is $0 until you add a test state registration.
- **The AI must never confirm a return itself.** The model has no confirm tool, and the confirmation token never enters model context or saved graph state.
- **LiveKit's OTel export covers traces only.** Turn its per-stage timing events into metrics with a small handler.
- **Cartesia:** pin `sonic-3.6-2026-08-27`. The LiveKit plugin otherwise defaults to `sonic-3`.
