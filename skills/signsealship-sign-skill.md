---
name: Sign
description: Use when building integrations for document signing, notarization, and shipping workflows; creating verified closing rooms for real estate transactions; sealing and verifying documents with cryptographic evidence; managing webhooks for order and room events; or implementing verification endpoints for sealed documents.
metadata:
    mintlify-proj: sign
    version: "1.0"
---

# Sign Seal Ship Skill

## Product summary

Sign Seal Ship is a pay-per-use API platform for getting documents signed, notarized (including remote online notarization), and shipped — with cryptographically verifiable evidence. The platform provides two core workflows: the **Order API** for creating individual sign/notarize/ship orders, and **Verified Closing Rooms** for bundling multiple documents into a single shareable transaction page with a sealed **Closing Passport** evidence manifest.

Key endpoints and concepts:
- **Partner API key** (`sss_pk_` prefix): Bearer token for all authenticated requests
- **Orders**: `POST /api/partner/orders` (create), `GET /api/partner/orders` (list), `GET /api/partner/orders/{code}` (fetch), `POST /api/partner/orders/{code}/checkout` (mint Stripe session)
- **Closing Rooms**: `POST /api/rooms` (create), `GET /api/rooms` (list), `POST /api/rooms/{roomCode}/orders` (attach), `DELETE /api/rooms/{roomCode}/orders/{orderCode}` (detach), `POST /api/rooms/{roomCode}/rotate` (rotate bearer link), `POST /api/rooms/{roomCode}/passport` (seal)
- **Webhooks**: Subscribe to `passport.sealed`, `room.order_attached`, `room.passport_sealed`, `order.created`, `payment.cleared`, `signature.completed`, `shipment.delivered`
- **Verification**: Public endpoints at `/v/{verifyCode}` (documents) and `/v/room/{verifyCode}` (passports) require no authentication

Primary docs: https://docs.signsealship.com

## When to use

Reach for this skill when:
- Building a partner integration to create orders programmatically (sign/notarize/ship workflows)
- Creating closing transaction pages that bundle multiple documents with live status tracking
- Sealing documents with cryptographic evidence and generating public verification URLs
- Implementing webhook handlers to react to order state changes, room events, or passport sealing
- Verifying sealed documents or Closing Passports without trusting SignSealShip
- Managing room bearer links and rotating them for security
- Pricing orders server-side with automatic partner-tier discounts applied
- Integrating Stripe hosted checkout for payment collection

Do not use this skill for: authentication configuration, dashboard-only operations, account creation, or pricing/billing management.

## Quick reference

### Authentication
- All partner endpoints require `Authorization: Bearer sss_pk_your_key`
- Keys are shown exactly once at issuance; store in a secrets manager
- Public verification endpoints (room links, verify codes) require no authentication
- Room links and verify codes are bearer credentials — treat like passwords

### Rate limits
| Policy | Limit | Applies to |
|--------|-------|-----------|
| `partner-write` | 60/min per key | Orders, rooms, passports |
| `public-read` | 30/min per IP | Room view, passport verification |
| `partner-request` | 5/hour per IP | Access requests |
| `public-write` | 12/min per IP | Proof Passport seal, verify |
| `partner-portal` | 120/min per session | Dashboard, webhook management |

### Order creation fields
- `document` (file) or `fill_token` (string) — exactly one, never both
- `email` (required), `name`, `signer_state`, `dest_state`
- `svc_sign`, `svc_notary`, `svc_ship` — service flags
- `byod_confirmed=true` (required) — attests client's own document
- `external_reference` — your matter/file number, echoed on webhooks
- `create_checkout=true` — mint Stripe session in same call

### Room creation fields
- `name` (1-200 chars, required) — display name shown to all parties
- `reference` — your internal file number, shown on room header

### Webhook signature verification
- Header: `SignSealShip-Signature: t={timestamp},v1={hex_signature}`
- HMAC key: `lowercase_hex(sha256(raw_secret))`
- Signed payload: `{t}.{rawBody}` (raw request bytes, not parsed JSON)
- Reject timestamps older than 5 minutes
- Use constant-time comparison for signature

### Evidence vocabulary
| Badge | Meaning | Verifiable without SignSealShip? |
|-------|---------|----------------------------------|
| **SEALED** | Cryptographic artifact hash on record | Yes — KMS signature, RFC 3161 timestamp, OpenTimestamps anchor |
| **RECORDED** | Platform attestation only | No — our word, not independent evidence |
| **No evidence yet** | Nothing to show | — |

## Decision guidance

### When to create an order vs. attach to a room
| Scenario | Use |
|----------|-----|
| Single document, client pays directly | Create order, mint checkout, hand Stripe URL to payer |
| Multiple documents in one transaction | Create separate orders, attach all to one room, share room link |
| Need to track multiple signers/services | Create order per document, attach to room for unified view |
| Want live status page for all parties | Always attach to a room, even single-document deals |

### When to seal a Closing Passport
| Scenario | Seal |
|----------|------|
| Snapshot at key milestone (all signed, ready to ship) | Yes — creates dated, versioned evidence |
| Every time an order status changes | No — quota is 20 versions per room; be intentional |
| Before sharing with external parties (lenders, title) | Yes — gives them independently verifiable proof |
| For internal tracking only | No — room JSON is sufficient |

### Bearer link security: rotate vs. share
| Situation | Action |
|-----------|--------|
| Link shared with correct parties, no leak | Keep using it |
| Link accidentally posted publicly or emailed to wrong person | Rotate immediately with `POST /api/rooms/{roomCode}/rotate` |
| Need per-party scoped access | Not yet available; rotation is current containment tool |
| Want to revoke access to all parties | Rotate the link; old link dies instantly at all cache layers |

## Workflow

### Create and seal a closing room

1. **Request access** — POST to `/api/partner/request` or use the form at signsealship.com/partner with firm name, work email, and use case. Business email + coherent use case = instant trial key; others go to human review.

2. **Receive your key** — Check email for `sss_pk_` key. Store it in a secrets manager immediately; it is shown exactly once.

3. **Create a room** — POST `/api/rooms` with `name` and optional `reference`. Capture the `roomCode` (40-char hex bearer token).

4. **Create orders** — For each document, POST `/api/partner/orders` with the PDF (or fill_token), email, services (`svc_sign`, `svc_notary`, `svc_ship`), and `byod_confirmed=true`. Capture each `orderCode`.

5. **Attach orders to room** — POST `/api/rooms/{roomCode}/orders` with each `orderCode`. Attaching is idempotent; a room holds up to 50 orders.

6. **Share the room link** — Give every party `https://signsealship.com/rooms/{roomCode}`. No accounts, no logins. The link is a bearer credential — share only with deal parties.

7. **Monitor progress** — Poll `GET /api/rooms/{roomCode}` for live status, or subscribe to `room.order_attached` and `room.passport_sealed` webhooks.

8. **Seal a Closing Passport** — When you want a verifiable snapshot, POST `/api/rooms/{roomCode}/passport`. Returns `verifyCode` and `verifyUrl` — anyone can verify at that public URL without trusting SignSealShip.

9. **Verify the passport** — Fetch `GET /api/verify/room/{verifyCode}` to get the manifest, hashes, and chain status. Re-canonicalize the manifest, re-hash it, and compare against `manifestSha256`. Walk the chain by fetching prior versions. Download the sealed PDF and verify the KMS signature in a standard PDF viewer.

### Handle webhooks

1. **Register endpoint** — POST `/api/partner/webhooks` with your HTTPS URL and topic list. Capture the `signingSecret` (`sss_whsec_` prefix) immediately; it is shown exactly once.

2. **Derive the HMAC key** — At startup, compute `lowercase_hex(sha256(raw_secret))` and store it. Do not re-derive per request.

3. **On delivery** — Read the raw request body bytes (do not parse JSON first). Extract `t` and `v1` from the `SignSealShip-Signature` header.

4. **Verify signature** — Compute HMAC-SHA256 over `{t}.{rawBody}` with the derived key. Compare against `v1` with constant-time comparison. Reject if `t` is older than 5 minutes.

5. **Parse and process** — Only after verification passes, parse the JSON and handle the event. Deduplicate on the `id` field (deterministic per event).

6. **Respond fast** — Return 2xx within 10 seconds. Process asynchronously; deliveries never block the operation that triggered them.

## Common gotchas

- **Storing the API key in code or version control** — Store in a secrets manager only. If leaked, contact SignSealShip immediately for deactivation.

- **Parsing JSON before verifying webhook signature** — Always verify against the raw request body bytes exactly as received. Re-serializing JSON changes the bytes and breaks the signature.

- **Assuming room links are private** — They are bearer credentials. One link grants deal-wide visibility. Never embed in a page, post publicly, or share with non-parties. Rotate if it leaks.

- **Sending both `document` and `fill_token` to order creation** — Exactly one, never both. The API rejects the request with a 400.

- **Forgetting `byod_confirmed=true` on order creation** — Required field. Attests the document is the client's own completed work (bring-your-own-document).

- **Attaching the same order to multiple rooms** — Idempotent, but the order appears in all rooms. Detach from one room before attaching to another if you want exclusive membership.

- **Expecting order state to change on checkout mint** — Order state advances only via verified Stripe webhook. Minting a checkout session or landing on the success page never changes state. Poll or subscribe to webhooks.

- **Treating RECORDED as independent evidence** — RECORDED is platform attestation only. If you need a claim to survive scrutiny beyond trusting SignSealShip, get it SEALED.

- **Assuming unknown order statuses are complete** — Unknown statuses render as *open*, never as complete. New states fail toward "less claimed".

- **Hitting room quota without realizing it** — Trial tier: 5 open rooms. PartnerLink: 5. ProOffice: 25. ClosingDesk: unlimited. Quota counts open rooms only; close a room to free the slot.

- **Sealing a passport every time an order changes** — Quota is 20 versions per room. Seal intentionally at key milestones, not on every status change.

- **Forgetting to store webhook signing secrets immediately** — Shown exactly once at creation. If lost, rotate with `POST /api/partner/webhooks/{id}/rotate` to get a new one (24-hour dual-signature overlap for cutover).

## Verification checklist

Before submitting work with Sign Seal Ship:

- [ ] API key is stored in a secrets manager, not in code or version control
- [ ] All authenticated requests include `Authorization: Bearer sss_pk_...` header
- [ ] Room links are shared only with deal parties, never posted publicly
- [ ] Webhook signature verification uses raw request body bytes, not parsed JSON
- [ ] Webhook handler is idempotent (deduplicates on `id` field)
- [ ] Webhook handler responds with 2xx within 10 seconds
- [ ] Order creation includes `byod_confirmed=true` and exactly one of `document` or `fill_token`
- [ ] External references are set on orders for webhook correlation
- [ ] Closing Passport is sealed only at intentional milestones, not on every status change
- [ ] Sealed documents are verified by re-hashing and comparing against `sealedSha256`
- [ ] Passport manifests are re-canonicalized (sorted keys, no whitespace) before re-hashing
- [ ] Evidence badges (SEALED, RECORDED, no evidence yet) are used correctly in UI
- [ ] Public verification URLs are tested with a fresh browser session (no auth required)
- [ ] Rate limits are respected (60/min for partner-write, 30/min for public-read)

## Resources

**Comprehensive page navigation:** https://docs.signsealship.com/llms.txt

**Critical documentation:**
- [Quickstart](https://docs.signsealship.com/quickstart) — Request access, create a room, attach orders, seal a passport in curl
- [Closing Rooms](https://docs.signsealship.com/closing-rooms) — Bearer links, rotation, lifecycle, tier quotas
- [Closing Passport](https://docs.signsealship.com/closing-passport) — Versioned manifests, hash chains, public verification, coverage statements
- [Order API](https://docs.signsealship.com/api-reference/orders) — Create, list, fetch orders; mint checkout sessions
- [Webhooks Guide](https://docs.signsealship.com/webhooks-guide) — Subscribe, verify signatures, handle events in Node and Python
- [Evidence Model](https://docs.signsealship.com/evidence-model) — SEALED vs. RECORDED, verification without trust
- [Authentication](https://docs.signsealship.com/api-reference/authentication) — Key issuance, rotation, rate limits

---

> For additional documentation and navigation, see: https://docs.signsealship.com/llms.txt