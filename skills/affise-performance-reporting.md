---
name: Pull Affise performance statistics
description: >-
  Query the Affise Performance API for conversion, revenue and traffic statistics across
  any dimension, then drill from an aggregate slice into the individual conversion records
  behind it.
api: openapi/affise-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/affise-openapi.yml + https://help-center.affise.com/en/articles/6796615-api-for-statistics-admins
operations:
  - GET /3.0/stats/custom
  - GET /3.0/stats/conversions
  - GET /3.0/stats/conversionsbyid
  - GET /3.0/stats/getbytrafficback
  - GET /3.0/stats/retentionrate
  - GET /3.0/stats/time-to-action
  - GET /3.0/admin/partners
  - GET /3.0/admin/advertisers
---

# Pull Affise performance statistics

> **Operation identifiers.** The Affise OpenAPI declares `operationId` on only 2 of its
> 167 operations, so every step below is identified by `METHOD path` — the only stable
> handle this contract offers. Each one is verbatim from `openapi/affise-openapi.yml`.

## Before you start

- **Base URL is per tenant.** Affise gives every customer its own host,
  `https://api-<company>.affise.com`. Read it from the admin panel under
  *Settings > Settings > Tracking domains > Default URL*. `api.affise.com` serves the
  documentation only — calling it with tenant data will not work.
- **Authenticate with `API-Key` in the header.** Send `API-Key: <key>` on every request.
  The same key is accepted as an `API-Key` query parameter; do not use that form —
  credentials in a URL end up in proxy logs, CDN logs and browser history.
- **The key carries a role, not a scope.** An admin key sees everything its user's
  platform role sees. There is no read-only credential, so a reporting agent given a
  General manager key can also call every write operation. Provision the narrowest role
  that satisfies the report.
- Read `conventions/affise-conventions.yml` before your first call.

## Step 1 — Choose the aggregate slice

Use **`GET /3.0/stats/custom`**. This one operation replaces the twenty-odd
`/3.0/stats/getby*` endpoints: pass the dimensions you want in `slice[]` and the measures
in `fields[]`.

- Always set `filter[date_from]` and `filter[date_to]` (`YYYY-MM-DD`).
- Always set `timezone` explicitly. Statistics are timezone-sensitive and the default is
  the tenant's, not yours — an unqualified "yesterday" will not reconcile against a
  report run in another zone.
- Narrow with the bracketed filters: `filter[offer]`, `filter[partner]`,
  `filter[advertiser]`, `filter[country]`, `filter[os]`, `filter[currency]`,
  `filter[sub1]`…`filter[sub5]`, `filter[offer_tag]`, `filter[affiliate_tag]`,
  `filter[advertiser_tag]`.
- Order with `order` plus `orderType` (`asc`/`desc`).

## Step 2 — Page through the result

Pagination is offset-based: `page` (1-based) and `limit`. The response carries a
`pagination` object of `{page, limit, total}` beside the data array.

There is **no cursor**. Over a high-churn statistics window, rows can shift between page
requests. For anything you intend to reconcile, pin `filter[date_from]`/`filter[date_to]`
to a closed period rather than paging through "today".

## Step 3 — Drill into the conversions

When a slice looks wrong, pull the event-level records with
**`GET /3.0/stats/conversions`** using the same filters. Fetch a single record with
**`GET /3.0/stats/conversionsbyid`**.

Conversion `status` values are numeric and mean:

| Value | Meaning |
|---|---|
| 1 | Approved |
| 2 | Pending |
| 3 | Declined |
| 5 | Approved and put on Hold |

Aggregate revenue that does not match a payout almost always comes down to which of these
statuses each side counted. State the status filter in every figure you report.

## Step 4 — Add the diagnostic slices

- **`GET /3.0/stats/getbytrafficback`** — traffic Affise redirected away instead of
  accepting, broken down by reason, partner and geo. A spike here is the usual cause of a
  click/conversion gap.
- **`GET /3.0/stats/retentionrate`** — retention across periods.
- **`GET /3.0/stats/time-to-action`** — the click-to-conversion delay distribution. A
  lengthening tail means today's conversions are still arriving for last week's clicks, so
  a same-day number will always understate.

## Step 5 — Resolve ids to names

Statistics rows carry ids, not labels. Resolve them with **`GET /3.0/admin/partners`** and
**`GET /3.0/admin/advertisers`**, both paginated the same way. Advertiser ids are MongoId
hex strings; affiliate, offer and user ids are integers — do not assume an id's type from
its value.

## Handling failures

Every response is a flat envelope with a numeric `status`: `1` is success, `2` is error.
There are no error codes — only an HTTP status and an English `message`.

| HTTP | `message` | What to do |
|---|---|---|
| 400 | Bad Request | Malformed request. Check URL encoding on long filter strings. |
| 401 | Token is necessary | Key missing, invalid, or aimed at the wrong tenant host. |
| 403 | Auth Denied | Key is valid; the user's role lacks permission for this resource. |
| 404 | Resource Not Found | Bad path or id. |
| 500 / 502 | Server error / Bad Gateway | Retry, then contact support@affise.com. |

**No rate limits are published**, no `RateLimit-*` or `Retry-After` headers are returned,
and no `429` is declared on any operation. Do not rely on a runtime backoff signal that
does not exist — self-throttle. Statistics queries are computationally expensive; keep
date ranges tight and prefer one `slice[]` query over many `getby*` calls.

Reads in this skill are safe to retry. Nothing here writes.
