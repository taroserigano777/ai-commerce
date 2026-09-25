# Acceptance, tests and evaluation

## Required automated checks

| Area | Cases that must pass |
|---|---|
| Identity | Customer A cannot list, read, quote, confirm or view a return for customer B; guest cannot see private orders; staff role is separate |
| Money | Server ignores client-submitted price/refund amount; currency arithmetic is exact; tax/shipping assumptions are documented |
| Stock | Concurrent checkouts cannot oversell; expired reservations release stock; price/stock change creates a new quote |
| Payment | Verified signature, duplicate/out-of-order webhooks, provider timeout/reconciliation, payment failure, refund pending/failure |
| Return | Ineligible category/date/quantity, repeat requests, stale or altered quote, quantity already returned, ambiguous item, required inspection |
| Confirmation | Exact item/amount/method bound to token; unknown or interrupted speech never confirms; repeated clicks yield same operation |
| Conversation | Policy answer citation; no policy hallucination when source absent; no cross-user conversation access; voice/text turn race controlled |
| Reliability | Worker retry after side effect, voice disconnect/resume, search index lag, cache eviction, model/provider timeout and human handoff |
| UI | Keyboard navigation, labels, focus, error/empty/loading states, responsive layout, captions and text fallback |

Use unit tests for domain rules, integration tests for database/workflow/provider adapters, and Playwright for the complete browser journey. Keep provider contract tests separate from simulators. A meaningful end-to-end test buys an item with Stripe test mode and returns it; when keys are unavailable, run the simulator path and label it.

## AI evaluation set

Create at least 200 labeled scenarios before a wider pilot, with examples across: simple policies, order status, one/multiple matching items, accents/noise, corrections, pauses, expired policies, exception requests, prompt injection in product/policy text, different customers, repeat confirmation, disconnection, failed tools, and status changes during conversation. Record expected intent, tool calls, identity boundary, policy evidence, user-visible response and whether handoff is needed.

Evaluate intent accuracy, return-item selection, policy citation correctness, false confirmations, unauthorized tool attempts, resolution rate, handoff rate, end-of-speech to meaningful audio, and cost per resolved task. Set a release threshold from a reviewed baseline; do not invent a success percentage from a small demonstration. Jev must be compared with the baseline router on the same set, including uncertainty calibration and fallback behavior.

## Proposed pilot service targets

These are targets to test, not claims of delivered performance:

- Product search p95 below 300 ms at the service boundary.
- Product page p75 Largest Contentful Paint below 2.5 s in real browser telemetry.
- Ordinary voice response: end of speech to first meaningful audio p50 below 1 s, p95 below 2 s.
- Tool-assisted voice response: end of speech to substantive answer p95 below 4 s when dependencies are healthy; report acknowledgments separately.
- Interruption: speech onset to stopped playback p95 below 300 ms.
- Commerce availability objective: 99.9% for the defined pilot request class.
- Zero unauthorized or duplicate refunds in adversarial release tests. Monitor and reconcile in production as well.

Load-test assumptions: 100k variants, 100k registered shoppers, 500 peak dynamic API requests/s, 20 checkout attempts/s and 50 simultaneous voice sessions. The build report must state hardware, data volume, provider quotas, actual achieved rate, p50/p95/p99, errors, and bottleneck. Do not imply the target was met without measurement.

## Human release review

Before a live pilot, a responsible owner must set real return rules, tax/shipping treatment, fulfillment provider behavior, merchant identity, privacy/retention policy, payment capture timing, fraud/manual-review criteria, support escalation, and production secrets. A coding agent may provide defaults for a sandbox only.
