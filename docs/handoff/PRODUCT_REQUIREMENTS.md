# Product requirements

## Product

Build an original US retail storefront with catalog, search, cart, checkout, account orders, returns, and a voice/text support assistant. Use original branding and licensed/sample product assets. The purpose is a realistic portfolio-quality application and a path to a limited production pilot, not an immediate nationwide retailer.

## People and permissions

- Guest: browse/search and read public help; may create a temporary cart. Must sign in or verify a guest order before seeing private order data.
- Customer: own cart, checkout, order history/status, return quote and confirmation, support chat/voice.
- Staff: review exceptional returns and support cases; inspect audit trail under a separate role.
- Administrator: manage catalog and approved policy versions in later milestone.

Enforce ownership and role checks on the server for every sensitive endpoint. A client-supplied customer ID is never authorization.

## Core journeys

### Shopping

Filter/search products by title, brand, category, price, and in-stock status. Product page shows variant, images, description, current price and availability. Cart survives refresh and sign-in. At checkout, the server recalculates price, discounts, tax/shipping quote, and stock; reserve inventory for the payment window. Redirect or embed Stripe's hosted payment collection. After a verified payment event, show an order with line items and fulfillment status. Handle failed or expired payment without creating a false paid order.

### Returns

The customer can start from the order page or say, “I want to return the headphones from last week.” If several items match, show cards and clarify. The backend checks owner, policy version, delivered date, category, quantity, previous returns, condition, and review flags. Present a structured quote with item, quantity, refund amount/currency, any deductions, method, destination, inspection requirement and expiry. Require explicit confirmation bound to that quote. Create one return and a durable operation ID. Show label/drop-off information only after authorization. Track refund separately and use accurate PENDING/SUCCEEDED/FAILED language. Manual review handles disputed or exceptional cases.

### Help and voice

Public policy questions use approved versioned content and show citations. Current price/availability, tracking, and refund status use authorized live APIs. Voice UI has mic permission, start/stop/mute, captions, interruption, connection status, and text fallback. When the user corrects “the other headphones,” stop speech and ask/select the intended item. A dropped voice session can resume the conversation and read the latest return status. Background speech or an uncertain transcript must never trigger confirmation.

## Pages

Home/category, search results, product detail, cart, checkout/status, account orders, order detail/returns, support panel, basic staff support queue, and accessible error/empty/loading states. The support panel stays usable on mobile. Show a clear indication when sandbox payments or simulated shipping is in use.

## Pilot scope

First-party US products, English, USD, one stock location and one shipping/label adapter, responsive web. Seed a modest original catalog and policy set. Do not build marketplace payouts, real-time store pickup, international taxes, loyalty, personalization, native mobile apps, or nationwide fulfillment in this release.

## Product outcomes

Measure completed purchase rate, return completion without staff, correct item selection, policy answer accuracy, duplicate/unauthorized financial action rate, voice task completion, handoff rate, accessibility, latency, and cost per resolved conversation. See ACCEPTANCE_AND_EVALS.md for proposed engineering targets.
