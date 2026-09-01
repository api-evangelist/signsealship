---
name: run-a-verified-closing
description: >-
  Create a Verified Closing Room, attach the orders that belong to a closing, add the people on
  the deal, share the bearer link, then seal a hash-chained Closing Passport that anyone can
  verify without an account. Use for a real-estate or title closing that bundles several
  documents into one shareable record.
api: SignSealShip Partner API
generated: '2026-09-01'
method: generated
source: openapi/signsealship-partner-api-openapi.yml
grounding: >-
  Every call below is a method+path pair read out of the published OpenAPI 3.1 document. The spec
  declares NO operationIds on any of its 39 operations, so method+path is the only stable
  identifier available; none has been invented.
operations:
  - POST /api/rooms
  - GET /api/rooms
  - POST /api/rooms/{roomCode}/orders
  - GET /api/rooms/{roomCode}/participants
  - POST /api/rooms/{roomCode}/participants
  - GET /api/rooms/{roomCode}
  - POST /api/rooms/{roomCode}/passport
  - GET /api/verify/room/{verifyCode}
  - POST /api/rooms/{roomCode}/rotate
---

# Run a Verified Closing

## Before you start

- Authenticate every partner call with `Authorization: Bearer sss_pk_...`. A missing, malformed,
  revoked or unknown key all return the same `401 {"error": "A valid partner API key is required."}`,
  so the body will not tell you which went wrong.
- Closing Rooms and passports live behind the `partner-write` limit: **60 requests per minute per
  key**. No rate-limit response headers are returned, so pace yourself rather than watching for a
  budget.
- Build against a `sss_pk_test_` key first. Test-environment evidence is marked `test` on every
  surface including the public verifier, so it can never be mistaken for production evidence.

## 1. Create the room

`POST /api/rooms` with `name` and your own `reference`.

Returns `roomCode`, `roomUrl`, `status`, `createdAt`. **Your tier's active-room quota is enforced
here** — 5 open rooms on trial and Partner Link, 25 on Pro Office, unlimited on Closing Desk. Over
quota is a `409`, not a `400`; do not retry it, close a room or upgrade.

## 2. Attach the orders

`POST /api/rooms/{roomCode}/orders` with `orderCode` and an optional `label`.

- Authorization is **possession of the order's public code**, not the room's.
- The call is idempotent: re-attaching an order that is already there succeeds with
  `alreadyAttached: true`. Treat that as success, not as an error.
- A room holds at most **50 orders**; the 51st is a `409`.
- Reversal: `DELETE /api/rooms/{roomCode}/orders/{orderCode}`. No window is stated.

## 3. Add the people on the deal

`GET /api/rooms/{roomCode}/participants` then `POST /api/rooms/{roomCode}/participants` with
`name`, `email`, `role`.

- Idempotent on a live email address, case-insensitively — a repeat returns `alreadyPresent: true`.
- At most **25 participants** per room.
- **Read this before removing anyone.** `DELETE /api/rooms/{roomCode}/participants/{participantId}`
  takes the person off the roster but does **not** revoke a room link they already hold. The
  participant list is a roster, not an access-control list. To actually cut off access, rotate the
  room code (step 6).

## 4. Share the link, and treat it as a credential

`GET /api/rooms/{roomCode}` is **public** — no key. Anyone holding the 40-hex room code can read
the live room JSON, which is exactly the same payload the room page renders: `progress`, every
attached `orders[]` card with `statusGroup` and `evidenceBadge`, `activity[]`, and `passport`
(the latest sealed version, if any).

Consequences you must design around:

- The room link **is** the bearer credential. Send it the way you would send a password.
- Public reads are limited to **30 per minute per IP** and served `Cache-Control: no-store`. Poll
  no more than once every few seconds per room; the provider's own page refreshes every 15 seconds.
- Build your logic on `statusGroup` (`signing`, `signed`, `notarizing`, `notarized`, `shipping`,
  `shipped`, `delivered`), not on the raw `status` string. The spec says so explicitly, and raw
  status values have already changed once.
- Rather than polling, subscribe to `room.order_attached` and `room.passport_sealed` — see
  `package-a-closing-passport` and `asyncapi/signsealship-webhooks.yml`.

## 5. Seal the Closing Passport

`POST /api/rooms/{roomCode}/passport`. Requires your partner key, and the room must be yours.

**This is irreversible and it is metered.** Sealing mints the *next* version — dated, hash-chained
and KMS-sealed — and never overwrites. There is no unseal and no delete. A room is capped at **20
versions**, so you have twenty seals for the life of the closing and no more. Seal at meaningful
milestones, not on every state change, and never in a retry loop.

Returns `version`, `verifyCode` (26 base32 chars), `verifyUrl`, `manifestSha256`,
`prevManifestSha256` (null on version 1 — that is the genesis link), `sealedSha256`.

Verify it back with `GET /api/verify/room/{verifyCode}`, which needs no key at all. `chainOk` is
**recomputed on every call** — the stored manifest is re-hashed and the chain link re-checked — so
it is a live proof, not a stored boolean. The sealed certificate PDF is at
`GET /api/verify/room/{verifyCode}/pdf`.

A malformed code and an unknown code both return an identical `404 {"verdict": "unknown"}`. That
sameness is deliberate: it stops the endpoint being used to enumerate which codes exist. Do not
interpret it as "the passport was deleted".

## 6. If a link leaks

`POST /api/rooms/{roomCode}/rotate` reissues the room's bearer code. **Every previously shared
link dies instantly and cannot be restored.** This is the correct and only mitigation for an
over-shared link — and it will break the link for everyone legitimately holding it, so re-send the
new one.

## Error handling

| Status | Meaning here | What to do |
| --- | --- | --- |
| 401 | Missing/invalid partner key | Fix the credential. Do not retry unchanged. |
| 403 | Valid session, no partner linked | Link a session with a partner key at signsealship.com/partner. |
| 404 | Unknown **or** not yours | Partner isolation is enforced through the 404. Never read it as "deleted". |
| 409 | Quota or state precondition | Room quota, 50-order cap, 25-participant cap, 20-version cap. Not retryable. |
| 429 | Rate limited | Back off to the next fixed one-minute window. No `Retry-After` is sent. |
| 502 / 503 | Upstream or platform | Retry with backoff; check signsealship.com/status. |

Every error body is `{"error": "<sentence>"}` — plain `application/json`, **not** RFC 9457. There
is no stable machine-readable error code, so branch on the HTTP status, never on the message text.
