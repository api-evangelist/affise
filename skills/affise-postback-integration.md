---
name: Wire up Affise conversion tracking with postbacks and pixels
description: >-
  Register the affiliate postback URLs Affise fires on every conversion, choose the right
  macro set, verify delivery, and fall back to pixels or conversion import when a
  server-to-server callback is not available.
api: openapi/affise-openapi.yml
generated: '2026-08-13'
method: generated
source: >-
  openapi/affise-openapi.yml +
  https://help-center.affise.com/en/articles/6466563-postback-integration-s2s-admins
operations:
  - POST /3.0/partner/postback
  - POST /3.0/partner/postback/{id}
  - DELETE /3.0/partner/postback/{id}/remove
  - DELETE /3.0/partner/postbacks/by-offers
  - DELETE /3.0/partner/postbacks/by-affiliates
  - GET /3.0/admin/postbacks
  - GET /3.0/partner/pixels
  - POST /3.0/partner/pixel
  - POST /3.0/partner/pixel/{id}
  - DELETE /3.0/partner/pixel/{id}/remove
  - GET /3.0/stats/affiliatepostbacks
  - GET /3.0/stats/serverpostbacks
  - POST /3.0/admin/conversion/import
  - POST /3.0/admin/conversions/import
---

# Wire up Affise conversion tracking with postbacks and pixels

> Identified by `METHOD path` — the Affise contract declares `operationId` on only 2 of 167
> operations. Full event-surface detail lives in
> `asyncapi/affise-postbacks-webhooks.yml`.

## The model

Conversion tracking runs in two directions and it helps to keep them apart:

- **Inbound** — the advertiser's server calls an Affise-generated postback URL to report
  that a user converted. Configured per offer in the UI.
- **Outbound** — Affise calls the *affiliate's* postback URL to pass that conversion on.
  This is the webhook, and it is the half you manage through the API.

## Before you start

- Tenant host `https://api-<company>.affise.com`, `API-Key` header, form-encoded bodies.
- Note the naming: every path says `partner`, every summary says `affiliate`. Same entity.

## Step 1 — Decide global or local

- **Global postback** — one URL per affiliate, fires for any offer.
- **Local postback** — per affiliate *per offer*, and it can additionally filter on goal
  value.
- **Consider global postback** — a fallback flag. When a conversion does not match the
  local postback's status, integration type or goal, the system re-checks the global
  postback. Without it, a conversion that misses the local filter is simply never
  delivered.

Register with **`POST /3.0/partner/postback`**; edit with
**`POST /3.0/partner/postback/{id}`**.

## Step 2 — Build the URL with macros

The postback URL is a template. Affise substitutes brace macros at fire time; there is no
JSON body — the payload *is* the query string.

A workable minimum:

```
https://partner.example/pb?cid={sub1}&txn={transactionid}&status={status}&payout={sum}&cur={currency}&offer={offerid}&ts={timestamp}
```

The fields that matter most:

| Macro | Why |
|---|---|
| `{sub1}`–`{sub8}` | **Where the click id lives.** There is no `{clickid}` macro on affiliate postbacks — Affise documents this explicitly. Map your click identifier here or conversions arrive unattributable. |
| `{transactionid}` | The advertiser's own conversion id. Your de-duplication key. |
| `{status}` | `1` Approved, `2` Pending, `3` Declined, `5` Approved-and-Hold. |
| `{sum}` / `{currency}` | Payout and its currency. |
| `{order_sum}` / `{order_currency}` | The basket value, distinct from the payout. |
| `{rand}` | A fresh UUID per fire — useful as a delivery id, **not** as a de-dup key. |
| `{timestamp}` / `{date}` / `{click_date}` | Conversion time, and the click that led to it. |

`{sub9}`–`{sub30}` exist only on higher plan tiers. Full vocabulary in
`asyncapi/affise-postbacks-webhooks.yml`.

## Step 3 — Build the receiver correctly

Three properties of this webhook decide how your endpoint must behave:

1. **No signature.** Affise sends no HMAC, no shared secret and no signing key. You cannot
   cryptographically verify that a conversion notification came from Affise, so anyone who
   learns your URL shape can forge one. Do not treat an incoming postback as authoritative
   for anything irreversible; reconcile against `GET /3.0/stats/conversions` before paying
   on it.
2. **Delivery is at-least-once.** Affise retries **five times with an incremental +2 second
   delay** on any 4xx or 5xx from your endpoint. Your handler must be idempotent — key on
   `{transactionid}`, not on `{rand}`.
3. **Return 200 fast.** Anything else starts the retry sequence. Acknowledge first, process
   asynchronously.

On the inbound leg the same asymmetry applies to *you*: Affise answers an inbound postback
with `status 1` **on receipt**, then validates in a queue. A 200 from Affise means
ACCEPTED, not APPLIED — the conversion can still be rejected afterwards.

## Step 4 — Verify delivery

- **`GET /3.0/stats/affiliatepostbacks`** — the outbound delivery log, including the HTTP
  code the affiliate's server returned. This is where you confirm a webhook is actually
  landing.
- **`GET /3.0/stats/serverpostbacks`** — the inbound advertiser postback log.
- **`GET /3.0/admin/postbacks`** — list registered affiliate postbacks.

Affise also documents guided integration tests for the S2S and pixel flows in the help
center; run one before going live.

## Step 5 — Clean up

- **`DELETE /3.0/partner/postback/{id}/remove`** — one postback.
- **`DELETE /3.0/partner/postbacks/by-offers`** and
  **`DELETE /3.0/partner/postbacks/by-affiliates`** — bulk removal. Both are destructive and
  neither is idempotent; enumerate with `GET /3.0/admin/postbacks` and confirm the target
  set immediately before calling.

## Fallbacks when S2S is not possible

- **Pixels (C2S)** — client-side tracking. `GET /3.0/partner/pixels`,
  `POST /3.0/partner/pixel` (the other operation carrying an `operationId`:
  `addPartnerPixel`), `POST /3.0/partner/pixel/{id}`,
  `DELETE /3.0/partner/pixel/{id}/remove`. Pixels carry a `moderation_status` — a newly
  created pixel is not necessarily live.
- **Conversion import** — `POST /3.0/admin/conversion/import` (single) and
  `POST /3.0/admin/conversions/import` (batch); affiliates use the `/4.0/` pair. **These
  have no idempotency key.** A retried batch import can double-post revenue. Set
  `transactionid` on every row and reconcile with `GET /3.0/stats/conversions` after import
  rather than retrying on timeout.

## Handling failures

`{"status": 1|2, "message": "..."}`; `401 Token is necessary`, `403 Auth Denied`,
`404 Resource Not Found`, `500`/`502`. No rate limits are published and no `429` is
declared. See `errors/affise-problem-types.yml`.
