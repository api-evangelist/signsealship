---
name: seal-and-verify-a-document
description: >-
  Seal an executed PDF into a Proof Passport with Google Cloud KMS, collect independent RFC 3161
  and OpenTimestamps evidence, and verify it later from anywhere with no account and no key. Use
  when you need a tamper-evident, independently checkable record of a finished document.
api: SignSealShip Partner API
generated: '2026-09-01'
method: generated
source: openapi/signsealship-partner-api-openapi.yml
grounding: >-
  Method+path pairs read from the published OpenAPI 3.1 document; the spec declares no
  operationIds.
operations:
  - POST /api/passport/seal
  - GET /api/passport/{id}
  - GET /api/passport/verify/{code}
  - GET /api/passport/verify/{code}/document
  - POST /api/passport/webhooks
  - DELETE /api/passport/webhooks/{id}
---

# Seal and verify a document

## What sealing actually is

`POST /api/passport/seal` takes a finished, signed PDF as the multipart part `file`, up to **35 MB**.
The bytes are streamed in memory, never written to disk, and discarded after sealing. What comes
back is a PDF carrying a CMS (PKCS#7) digital signature made with a non-exportable Google Cloud KMS
asymmetric key, covering the PDF's ByteRange — so it validates in Adobe Reader's signature panel on
the reader's own machine, with no SignSealShip account and no call to this API.

**Sealing is irreversible.** There is no unseal, no revoke, no delete. Rate limited under
`public-write`: 12 per minute per IP.

## What the response gives you

- `passportId` (uuid) — your private handle. Partner-isolated: only the partner that created a
  passport can fetch it.
- `verifyCode` / `verifyUrl` — the **public** handle. Anyone holding the code can verify the
  passport and download the sealed document. The code is a bearer credential; treat it as one.
- `docSha256` — SHA-256 of the exact bytes you uploaded, pre-seal.
- `sealedSha256` — SHA-256 of the sealed PDF that came back.
- `environment` — `live` or `test`, from the key that sealed it.
- `timestamps` — two independent authorities, either of which may be `null` when that authority
  could not be reached. Check for null rather than assuming both landed:
  - `timestamps.rfc3161` → `{authority, timestampedAtUtc}` (RFC 3161 Time-Stamp Protocol)
  - `timestamps.openTimestamps` → `{calendar, submittedAtUtc, status}` — `status` is often
    `pending` at seal time, because a Bitcoin-anchored attestation confirms later. A `pending`
    OpenTimestamps entry is normal and is not a failure.

Keep `docSha256` and `sealedSha256` in your own records. They are what lets you prove, later and
independently, that the document you hold is the document that was sealed.

## Verifying

- `GET /api/passport/verify/{code}` — **public, no key**. Returns `verdict`, `environment`,
  `verifyCode`, `sealedSha256`, `docSha256`, `sealedAtUtc`, `source`, `certificateAvailable`.
- `GET /api/passport/verify/{code}/document` — **public, no key**. Streams the byte-identical
  sealed PDF.
- `GET /api/passport/{id}` — partner-only fetch by uuid.

**Read the verdict carefully.** `verified` is production evidence. `verified_test` means the
passport was sealed with a TEST-environment key and **is not production evidence** — the public
verifier flags it precisely so sandbox output can never be laundered into a real record. Never
present a `verified_test` passport as proof of anything.

An unknown code and a malformed code both return an identical `404 {"verdict": "unknown"}`. The
sameness is intentional and stops the endpoint being used to discover which codes exist. It does
not mean a passport was deleted.

The public verification endpoints never expose the partner identity behind a passport.

## Getting told when a seal happens

`POST /api/passport/webhooks` registers an **https** endpoint for `passport.sealed` only.
Subscriptions created this way carry no topic list and never receive room or order events; for
those, use the topic-aware `/api/partner/webhooks` routes instead.

The signing secret is returned **exactly once** — only its hash is stored. Verify deliveries
against the `SignSealShip-Signature` header, formatted `t=<unix seconds>,v1=<hex hmac>`, with the
topic in `SignSealShip-Event`. The API will hand you the recipe itself at
`GET /api/partner/webhooks/signature-example`, and the docs publish working Node and Python
verifiers.

Remove one with `DELETE /api/passport/webhooks/{id}`.

## Two things that will bite you

1. **The 35 MB limit is on the multipart part**, and there is no chunked or resumable upload. Check
   the size before you send.
2. **`environment` is a property of the key, not of the request.** There is no flag to pass. If you
   want test evidence you must hold a `sss_pk_test_` key; if you hold a live key, everything you
   seal is real, permanent and publicly verifiable.
