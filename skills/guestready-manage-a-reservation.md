---
name: guestready-manage-a-reservation
description: Create a direct reservation, amend arrival time and key code, attach documents, and cancel it — the full booking lifecycle in RentalReady.
api: RentalReady API
base_url: https://pms.rentalready.io/api/v3/
generated: '2026-08-22'
method: generated
source: openapi/guestready-rentalready-openapi.yml
scopes:
  - reservations:read
  - reservations:write
  - rentals:read
operations:
  - reservations_list
  - reservations_create
  - reservations_retrieve
  - reservations_partial_update
  - reservations_update_arrival_time_partial_update
  - reservations_update_key_code_partial_update
  - reservations_cancel_partial_update
  - reservations_custom_fields_retrieve
  - reservations_custom_fields_create
  - reservation_documents_create
  - inquiries_list
---

# Manage a reservation

## Before you start

- `reservations_create` is **not idempotent** and there is no `Idempotency-Key`. A retried `POST /api/v3/reservations/` can double-book a property. If a create times out, call `reservations_list` with `created_after` set to one minute before the attempt and check before retrying.
- Reservations arriving from Airbnb / Booking.com / Vrbo are created by the channel sync, not by you. Use `reservations_create` for **direct** bookings only.
- All scopes are per-resource: reading needs `reservations:read`, every write needs `reservations:write`.

## Steps

1. **Check availability first.** `daily_prices_list` and the calendar tell you whether the dates are open; there is no atomic "book if free" operation, so a read-then-write race is possible on a busy property.
2. **Create** — `reservations_create` (`POST /api/v3/reservations/`). The body accepts `guest_invoice` (a nested `GuestInvoice`) and a `status` from `Status3eaEnum`. The response inlines `extra_fees`, `mid_term_period_breakdown`, `payment_links_deposits` and `voucher_redemption`.
3. **Read it back** — `reservations_retrieve` (`GET /api/v3/reservations/{id}/`). Do this before any amendment; several amendments are status-dependent.
4. **Amend.** Use the narrow operations rather than a broad PATCH where one exists — they carry the business rules:
   - `reservations_update_arrival_time_partial_update` (`PATCH /api/v3/reservations/{id}/update_arrival_time/`)
   - `reservations_update_key_code_partial_update` (`PATCH /api/v3/reservations/{id}/update_key_code/`) — rejects with `400 {"message": "invalid key_code"}` if the code does not match the format the property's smart lock expects
   - `reservations_partial_update` (`PATCH /api/v3/reservations/{id}/`) for everything else
5. **Attach documents** — `reservation_documents_create` (`POST /api/v3/reservation_documents/`), typed by `DocumentTypeEnum`.
6. **Custom fields** — `reservations_custom_fields_retrieve` / `reservations_custom_fields_create` on `/api/v3/reservations/{id}/custom_fields/`. Definitions come from `custom_fields_list`.

## Cancelling — the reversal path

`reservations_cancel_partial_update` (`PATCH /api/v3/reservations/{id}/cancel/`, scope `reservations:write`) is the reversal of a booking. It is a **status transition, not a delete** — the reservation stays readable afterwards.

**There is no published cancellation window.** The guest-facing refund terms live in the property's cancellation policy (`CancellationPolicyCategoryEnum`), not on this operation, and neither the spec nor the public Help Center states a deadline after which cancellation stops working. Do not assume one, and do not tell a user a booking "can still be cancelled" on the strength of anything in this repository — read the property's policy.

Money already taken is separate: refunding it is `payment_acceptance_transactions_refund_create`, and it is Stripe-only (see `guestready-reconcile-payments.md`).

## Errors

Only 11 of 252 operations declare a 4xx in the contract, and none declares 401 or 404 — treat both as possible on every call. Full catalogue: `errors/guestready-problem-types.yml`.
