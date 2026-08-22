---
name: guestready-reconcile-payments
description: Read invoicing and payment links, settle payout adjustments, and refund a payment acceptance transaction — with the preconditions the contract actually declares.
api: RentalReady API
base_url: https://pms.rentalready.io/api/v3/
generated: '2026-08-22'
method: generated
source: openapi/guestready-rentalready-openapi.yml
scopes:
  - reservations:read
  - guest_registration:read
  - payment_links:read
  - payout_adjustments:read
  - payout_adjustments:write
  - payment_acceptance_transactions:read
  - payment_acceptance_transactions:write
  - accounting_invoice:write
operations:
  - invoicing_list
  - invoicing_retrieve
  - payment_links_list
  - payment_links_invoice_request_retrieve
  - payout_adjustments_list
  - payout_adjustments_retrieve
  - payout_adjustments_invoice_request_retrieve
  - payout_adjustments_mark_as_paid_create
  - payment_acceptance_transactions_list
  - payment_acceptance_transactions_retrieve
  - payment_acceptance_transactions_refund_create
  - accounting_invoices_create
  - swikly_deposits_list
---

# Reconcile payments, payouts and refunds

> **This skill touches money and the API has no idempotency key.** Every `POST` here can be double-applied by a retry. Read state before you write, and never blind-retry a timeout.

## Read the financial picture

- `invoicing_list` (`GET /api/v3/invoicing/`) — needs **two** scopes, `reservations:read` **and** `guest_registration:read`. Returns `InvoicingReservation` with `extra_fees` and `guest_registration` inlined.
- `payment_links_list` (`GET /api/v3/payment_links/`, scope `payment_links:read`) — guest payment links with `status` and `category`.
- `payment_acceptance_transactions_list` (`GET /api/v3/payment_acceptance_transactions/`) — actual captures.
- `swikly_deposits_list` — security deposits held through Swikly, which are a separate rail from Stripe.
- `payout_adjustments_list` (`GET /api/v3/payout_adjustments/`) — owner payout corrections, with nested `invoices`.

## Settle a payout adjustment

`payout_adjustments_mark_as_paid_create` (`POST /api/v3/payout_adjustments/{id}/mark_as_paid/`, scope `payout_adjustments:write`).

**Read the adjustment first.** The contract declares `400 {"message": "Payout adjustment is already paid"}` — this operation is explicitly not idempotent, and the API tells you so by rejecting the second call rather than absorbing it. A missing permission returns `403 {"message": "Permission to mark payout adjustments as paid is required."}`.

There is **no un-mark operation**. Marking paid is one-way.

## Refund a payment

`payment_acceptance_transactions_refund_create` (`POST /api/v3/payment_acceptance_transactions/{id}/refund/`, scope `payment_acceptance_transactions:write`).

Two preconditions are declared in the contract itself:

- `400 {"message": "Refund not allowed for acceptance transaction"}` — not every acceptance transaction is refundable. Retrieve the transaction and check its state first.
- `403 {"message": "Refund allowed only for stripe payment gateway"}` — **refunds through this API are Stripe-only.** A payment taken through any other gateway must be reversed at source, outside RentalReady.

**No refund window is published.** Neither the specification nor the public Help Center states a deadline after which a refund stops being possible. Do not quote one. If a user needs to know how long they have, the answer must come from RentalReady or from Stripe, not from here.

## Raise an accounting invoice

`accounting_invoices_create` (`POST /api/v3/accounting_invoices/`, scope `accounting_invoice:write`) — typed by `AccountingInvoiceCreateCategoryEnum` / `AccountingInvoiceCreateTypeEnum` with a currency from `CurrencyD53Enum`. Note there is **no** list, retrieve, update or delete for accounting invoices: create is the only operation on this resource, so verify externally and get it right on the first call.

## Related

`conventions/guestready-conventions.yml` (reversibility, idempotency, pagination) · `errors/guestready-problem-types.yml` (every declared 4xx) · `scopes/guestready-scopes.yml` (all 54 scopes).
