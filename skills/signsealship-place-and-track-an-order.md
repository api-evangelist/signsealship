---
name: place-and-track-an-order
description: >-
  Create a sign / notarize / ship order for a client's completed document, mint the Stripe hosted
  checkout, and follow the order to delivery — safely, using the idempotency key and the sandbox
  simulation endpoints, and knowing that nothing in this API can undo a paid order.
api: SignSealShip Partner API
generated: '2026-09-01'
method: generated
source: openapi/signsealship-partner-api-openapi.yml
grounding: >-
  Method+path pairs read from the published OpenAPI 3.1 document. The spec declares no
  operationIds, so no operation identifier has been invented here.
operations:
  - GET /api/partner/me
  - POST /api/partner/orders
  - GET /api/partner/orders
  - GET /api/partner/orders/{code}
  - POST /api/partner/orders/{code}/checkout
  - POST /api/partner/sandbox/orders/{code}/checkout
  - POST /api/partner/sandbox/orders/{code}/advance
  - GET /api/partner/launchpad
  - GET /api/partner/analytics
---

# Place and track an order

## Read this first

**Nothing in this API reverses a paid order.** There is no refund, void, cancel or reverse
operation anywhere in the 39 published operations. Once the payer completes checkout, the verified
Stripe webhook advances the order and the money has moved; undoing it is a human process governed
by <https://signsealship.com/legal/refunds>. Rehearse in the sandbox before you spend.

The Order API is gated at the **$149/mo Pro Office** tier. Partner Link explicitly does not include
it. Confirm what your key can do with `GET /api/partner/me`, which returns `401` with no credential
and `403` for a valid session that has not been linked to a partner.

## 1. Create the order

`POST /api/partner/orders`, `multipart/form-data`.

Supply the document exactly one way: the PDF as the multipart part `document`, **or** a
fill-online result as `fill_token`. Never both.

Fields worth knowing: `email`, `name`, `doc_slug`, the service flags `svc_sign` / `svc_notary` /
`svc_ship`, `signer_state`, `dest_state`, `byod_confirmed`, `external_reference`, and the
`ship_*` address fields.

- `signer_state` and `dest_state` matter: state-and-document eligibility for remote online
  notarization is checked **before** payment and a restricted combination is refused rather than
  fudged. Look the rules up in advance at <https://signsealship.com/ron-laws.json> (free, CC BY
  4.0, statute-cited per value).
- `external_reference` is the only key you control. It is echoed on order responses, filterable on
  `GET /api/partner/orders?external_reference=`, and repeated on every order webhook event. Set it
  on every order — it is your correlation handle.

**Send an `Idempotency-Key` header.** It is optional in the spec and mandatory in practice for an
agent: 8–255 characters, one key per logical action. A retry carrying the same key replays the
original response instead of creating a second order. The spec does not state how long a key is
remembered, so do not rely on it days later.

Pricing is computed entirely server-side from the B2B price book with your tier discount applied —
you cannot set a price. The response carries `requestId` (quote it in support tickets),
`orderCode`, `orderUrl`, `status`, `subtotalCents` / `discountCents` / `totalCents`, and `lines[]`
of `{type, label, amountCents}`. Amounts are **integer cents with no currency field** — USD is
implicit.

Since 2026-08-01 shipping arrives as **two** lines, `ShippingCarrierRate` and
`ShippingHandlingFee`, and the postage amount depends on the destination address. Do not assume a
flat shipping line; that shape was removed.

## 2. Mint checkout

`POST /api/partner/orders/{code}/checkout` returns `checkoutUrl` — a Stripe hosted-checkout URL to
hand to the payer.

Minting is **state-neutral and safe to repeat**: it never advances the order, and neither does the
payer landing on the success page. State advances only via the verified Stripe webhook after
payment clears. Send the `Idempotency-Key` header here too.

## 3. Rehearse in the sandbox

With a `sss_pk_test_` key, same host, same paths:

- `POST /api/partner/sandbox/orders/{code}/checkout` with `{"outcome": "success" | "decline" |
  "cancel" | "delayed"}`. `success` finishes the payment crossing exactly like the real Stripe
  webhook and fires `payment.cleared` **with real HMAC signing**; a replayed success is a no-op.
  `decline` parks at PaymentFailed, `cancel` and `delayed` park at AwaitingPayment. No Stripe
  session is ever minted for a test order.
- `POST /api/partner/sandbox/orders/{code}/advance` with `{"event": null}` to cross whichever
  milestone is next, or name one of `payment.cleared`, `signature.completed`,
  `shipment.delivered`. `firedEvents[]` in the response names the webhooks it actually delivered.

Both return `409` for a live order, and `advance` also returns `409` for a milestone the purchased
services can never reach. There are no test card numbers to look up — the outcome enum replaces
them.

## 4. Track it

Prefer webhooks over polling. Subscribe to `order.created`, `payment.cleared`,
`signature.completed` and `shipment.delivered`; they deliver only for orders you created through
this API, echo your `external_reference`, and carry a deterministic id of `{topic}:{orderCode}` so
a redelivery deduplicates cleanly.

Polling fallback: `GET /api/partner/orders` (newest first, `limit` and `order` only — **no cursor,
no total**, so you cannot page past the limit or tell a truncated list from a complete one) and
`GET /api/partner/orders/{code}`.

Branch on `statusGroup`, never on `status`. The spec says so in as many words: `status` is the raw
platform value (e.g. `EsignPending`), `statusGroup` is the stable contract —
`signing` → `signed` → `notarizing` → `notarized` → `shipping` → `shipped` → `delivered`.

## 5. Check your posture

- `GET /api/partner/launchpad` — the go-live checklist.
- `GET /api/partner/analytics?days=` — a request-time snapshot over a 1–366 day window: volume,
  lifecycle funnel, paid-to-completed turnaround, service mix, revenue, and evidence counts.

## Errors

`404` means unknown **or** belonging to another partner — the two are deliberately
indistinguishable, so never treat it as proof of deletion. `429` is the `partner-write` limit, 60
per minute per key, with no `Retry-After` header to guide you. Every error body is
`{"error": "<sentence>"}`, not RFC 9457, so branch on the status code and never on the message.
