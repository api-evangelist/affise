---
name: Launch an Affise offer and hand a tracking link to an affiliate
description: >-
  Create an offer against an advertiser, categorise and target it, enable the affiliates
  who may run it, and generate the tracking link they promote.
api: openapi/affise-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/affise-openapi.yml + https://help-center.affise.com/en/articles/6795148-api-for-offers-admins
operations:
  - GET /3.0/admin/advertisers
  - GET /3.0/offer/categories
  - POST /3.0/admin/offer
  - POST /3.0/admin/offer/{id}
  - GET /3.0/offer/{id}
  - POST /3.0/offer/enable-affiliate
  - POST /3.0/offer/disable-affiliate
  - GET /3.1/offers/{id}/privacy
  - POST /3.0/admin/offer/{offerId}/tracking-link
  - GET /ui/offers/available-for/{affiliate_id}
---

# Launch an Affise offer and hand a tracking link to an affiliate

> Identified by `METHOD path`: the Affise contract declares `operationId` on only 2 of 167
> operations. Every path below is verbatim from `openapi/affise-openapi.yml`.

## Before you start

- Call your own tenant host, `https://api-<company>.affise.com`, with `API-Key` in the
  header. See `authentication/affise-authentication.yml`.
- **Write bodies are form-encoded, not JSON.** Send
  `Content-Type: application/x-www-form-urlencoded` (or `multipart/form-data` when
  uploading a logo or creative). Posting JSON will fail with `400 Bad Request`.
- **There is no idempotency key.** If `POST /3.0/admin/offer` times out, you cannot safely
  retry it — you may create a duplicate offer. Recover by listing offers and matching on
  `title` or `external_offer_id` before you try again.

## Step 1 — Pick the advertiser and the category

- **`GET /3.0/admin/advertisers`** — find the advertiser this offer belongs to. Its id is a
  MongoId hex string, not an integer.
- **`GET /3.0/offer/categories`** — get the category ids. Categories are tenant-defined;
  do not hardcode them across instances.

## Step 2 — Create the offer

**`POST /3.0/admin/offer`**

The `offer` schema carries 81 properties. The ones that decide whether the offer works:

- `title`, `advertiser`, `url` (the advertiser's destination, with your tracking macros),
- `trafficback_url` — where traffic goes when it fails targeting. Set it. Traffic that
  fails targeting with no trafficback URL is simply lost, and shows up later as an
  unexplained click/conversion gap in `GET /3.0/stats/getbytrafficback`.
- `external_offer_id` — your own identifier. Set it: it is the only handle that lets you
  detect a duplicate created by a retried request.

Structured sub-objects are declared as reusable schemas and can be sent with the offer:

| Field | Schema | Purpose |
|---|---|---|
| `payments` | `offer_payment_structure` | Payout per goal, country, device, OS, sub-id condition |
| `caps` | `offer_caps_structure` | Volume limits per period, goal, affiliate or country |
| `targeting` | `targeting_group_structure` | Geo, OS, ISP, IP, device, browser, connection, proxy blocking |
| `tiers` | `offer_tier_structure` | Payout modifiers above a performance threshold |
| `schedule` | `offer_schedule_structure` | Start/end dates, timezone, dayparting |
| `landings` | `offer_landing_structure` | Landing and prelanding pages |

Edit later with **`POST /3.0/admin/offer/{id}`** — note that Affise uses POST for both
create and update; there is no PUT or PATCH anywhere in the contract. The difference is the
path, not the method.

## Step 3 — Verify before you expose it

**`GET /3.0/offer/{id}`** and read back what was actually stored. Targeting groups and
payout structures are the fields most often silently dropped when a form-encoded nested
array is mis-serialised — confirm they came back, do not assume the `status: 1` meant they
landed.

## Step 4 — Choose who may run it

- **`POST /3.0/offer/enable-affiliate`** / **`POST /3.0/offer/disable-affiliate`** — grant
  or revoke access for specific affiliates.
- **`GET /3.1/offers/{id}/privacy`** — read back the enabled/disabled affiliate list. Do
  this after every change: the offer's privacy level and its per-affiliate list interact,
  and the privacy level alone does not tell you who currently has access.
- **`GET /ui/offers/available-for/{affiliate_id}`** — confirm the offer now appears for
  that affiliate. (This path is unversioned and is an internal UI endpoint exposed in the
  public reference — treat it as convenience, not contract.)
- Bulk operations exist: **`POST /3.0/admin/offer/{id}/disable-affiliates`** and
  **`POST /3.0/admin/affiliate/{id}/disable-offers`**.

## Step 5 — Generate the tracking link

**`POST /3.0/admin/offer/{offerId}/tracking-link`**

Pass the affiliate id and any sub-id values to bake into the link. This is the artefact the
affiliate actually promotes.

**Carry the click identifier in `sub1`–`sub8`.** Affise has no `{clickid}` macro on
affiliate postbacks — the documentation says so explicitly. If you are migrating from a
network that used a dedicated click-id field, map it to a sub parameter now, or conversions
will come back unattributable. Sub-ids 9 through 30 exist only on higher plan tiers.

## Step 6 — Change the offer's status

**`POST /3.0/admin/offer/mass-update`** updates offer status in bulk. Deleting is
**`POST /3.0/admin/offer/delete`** — a POST, not a DELETE.

## Handling failures

Responses are `{"status": 1|2, "message": "..."}`. `401 Token is necessary` means the key
is missing, invalid, or pointed at the wrong tenant. `403 Auth Denied` means the key is
valid but the role is not a General manager — only a General manager can perform the full
offer surface. No rate limits are published and no `429` is declared; self-throttle bulk
work.

See `errors/affise-problem-types.yml` and `conventions/affise-conventions.yml`.
