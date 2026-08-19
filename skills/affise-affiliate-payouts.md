---
name: Generate and settle Affise affiliate payouts
description: >-
  Reconcile approved conversions into payment invoices for affiliates, apply corrections,
  and close the billing loop against advertiser invoices — with the retry hazards called
  out, because this surface has no idempotency key.
api: openapi/affise-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/affise-openapi.yml + https://help-center.affise.com/en/articles/6244063-api-for-billings-admins
operations:
  - GET /3.0/stats/conversions
  - GET /3.0/admin/partners
  - POST /3.1/payments/generate
  - POST /3.1/payments/create-for-affiliate
  - GET /3.1/payments
  - GET /3.1/payments/{id}
  - POST /3.1/payments/{id}
  - POST /3.1/payments/{id}/add-correction
  - DELETE /3.1/payments/del-correction/{id}
  - POST /3.1/payments/{invoice_id}/conversions/delete
  - DELETE /3.1/payments/delete/{id}
  - POST /3.1/payments/bulk-delete
  - GET /3.0/admin/advertiser-invoices
  - POST /3.0/admin/advertiser-invoice
  - GET /3.0/admin/payment_systems
---

# Generate and settle Affise affiliate payouts

> **This skill moves money.** Nothing on this surface is idempotent — read the hazard
> section before the first call. Identified by `METHOD path`; the Affise contract declares
> `operationId` on only 2 of 167 operations.

## The hazard, first

Affise publishes **no `Idempotency-Key` header and no request de-duplication** on any write
operation. `POST /3.1/payments/generate` creates payout invoices from approved conversions.
If that request times out or the connection drops, **you do not know whether it ran**, and
retrying it can generate a second set of invoices for the same conversions.

The safe pattern is read-back, never blind retry:

1. Before generating, snapshot `GET /3.1/payments` for the target period.
2. Issue the generate call.
3. On timeout or any ambiguous response, **do not retry.** Re-read `GET /3.1/payments` and
   compare against the snapshot to establish what actually happened.
4. Only then decide whether to generate again, or to clean up with
   `DELETE /3.1/payments/delete/{id}` / `POST /3.1/payments/bulk-delete`.

Treat every operation in this skill as requiring human approval when an agent is driving.

## Before you start

- Tenant host `https://api-<company>.affise.com`, `API-Key` header. A **General manager**
  role is required; other roles will get `403 Auth Denied`.
- Write bodies are `application/x-www-form-urlencoded`, not JSON.
- `GET /3.0/admin/payment_systems` lists the payment systems configured on the tenant.

## Step 1 — Establish what is payable

**`GET /3.0/stats/conversions`** with `filter[date_from]`, `filter[date_to]` and an explicit
`timezone`.

Only **status 1 (Approved)** conversions are payable. Status 2 is Pending, 3 is Declined,
and **status 5 is "Approved and put on Hold"** — approved but deliberately withheld. Status
5 is the value that most often causes a payout dispute, because a naive filter for
"approved" that matches on the word rather than the numeric value can sweep held
conversions into a payment run. Filter on the number.

Resolve affiliate ids to names with **`GET /3.0/admin/partners`**.

## Step 2 — Generate the invoices

- **`POST /3.1/payments/generate`** — generate payment invoices for affiliates in bulk over
  a period. This is the high-risk call above.
- **`POST /3.1/payments/create-for-affiliate`** — create a single invoice for one affiliate.
  Prefer this when you can: the blast radius of an accidental double-run is one affiliate,
  not the whole book.

## Step 3 — Review

- **`GET /3.1/payments`** — list invoices, paginated with `page`/`limit`.
- **`GET /3.1/payments/{id}`** — one invoice. The `payment` schema carries `pid` (the
  affiliate), `manager_id`, `pay_sys` / `pay_sys_name`, `pay_acc`, `status`, `revenue`,
  `ref_revenue`, `currency`, `cpa_actions` (the conversions rolled up) and
  `additional_payments`.

Reconcile `revenue` against your Step 1 figure **for the same status filter and the same
timezone**. If they disagree, the difference is almost always status 5 or a timezone
boundary, not a platform error.

## Step 4 — Correct rather than delete

- **`POST /3.1/payments/{id}/add-correction`** — add a correction value to an invoice.
  Reversible with **`DELETE /3.1/payments/del-correction/{id}`**.
- **`POST /3.1/payments/{invoice_id}/conversions/delete`** — remove specific conversions
  from an invoice. (One of the only two operations in the whole contract that carries an
  `operationId`: `deleteInvoiceConversions`.)
- **`POST /3.1/payments/{id}`** — update the invoice.

Corrections leave an audit trail; deleting and regenerating does not. Prefer a correction.

## Step 5 — Delete only as recovery

**`DELETE /3.1/payments/delete/{id}`** and **`POST /3.1/payments/bulk-delete`** exist to
undo a bad run. Note the asymmetry: single delete is a `DELETE`, bulk delete is a `POST`.
Confirm the target ids against a fresh `GET /3.1/payments` immediately before calling
either — the API gives you no confirmation step and no undo.

## Step 6 — Close the advertiser side

- **`GET /3.0/admin/advertiser-invoices`** and
  **`GET /3.0/admin/advertiser-invoice/{number}`** — the receivable side.
- **`POST /3.0/admin/advertiser-invoice`** creates one;
  **`POST /3.0/admin/advertiser-invoice/{number}`** edits it.

Advertiser invoices are keyed by `{number}`, not by `{id}` — a different identifier
convention from the affiliate payments surface in the same API.

## Handling failures

`{"status": 1|2, "message": "..."}` with no error codes. `401 Token is necessary`,
`403 Auth Denied`, `500 Server error`. No `429` is declared, no rate-limit headers are
returned, and no `Retry-After` is sent — so there is no runtime signal telling you when a
retry is safe. On this surface, that means: do not retry writes automatically. Ever.

See `errors/affise-problem-types.yml` and `conventions/affise-conventions.yml`.
