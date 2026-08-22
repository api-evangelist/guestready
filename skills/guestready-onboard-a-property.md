---
name: guestready-onboard-a-property
description: Create a rental (property) in RentalReady, build out its layout, amenities, descriptions, photos and check-in steps, connect it to an OTA listing, then activate it.
api: RentalReady API
base_url: https://pms.rentalready.io/api/v3/
generated: '2026-08-22'
method: generated
source: openapi/guestready-rentalready-openapi.yml
scopes:
  - rentals:write
  - rentals:read
  - amenities:write
  - photos:write
  - owners:write
operations:
  - rentals_create
  - rentals_retrieve
  - rentals_partial_update
  - bedrooms_create
  - bathrooms_create
  - subrooms_create
  - rental_amenities_create
  - descriptions_create
  - photos_create
  - check_in_steps_create
  - check_in_steps_order_create
  - owners_create
  - listings_create
  - listings_enable_create
  - rentals_enable_create
---

# Onboard a property into RentalReady

## Before you start

- Every path ends in a **trailing slash**. `POST /api/v3/rentals` returns `400 {"detail": "Missing trailing slash..."}`; `POST /api/v3/rentals/` is the call.
- Auth is OAuth 2.0 authorization code. Exchange at `https://pms.rentalready.io/o/token/`; the authorization `code` is valid for **60 seconds**. Send `Authorization: Bearer <access_token>`.
- Confirm who you are first: `GET /api/v3/users/me/` (`users_me_retrieve`, scope `users:read`).
- **There is no idempotency key.** If a `POST` times out, do NOT blind-retry — `GET` the collection with `created_after` set to just before your attempt and check whether the object landed.
- Stay under **400 requests/minute**. No `RateLimit-*` header is returned, so meter yourself.

## Steps

1. **Create the owner if they do not exist yet** — `owners_create` (`POST /api/v3/owners/`, scope `owners:write`). Owners are keyed by `username`, not by an integer id. Note it; you will need it on the rental.
2. **Create the rental** — `rentals_create` (`POST /api/v3/rentals/`, scope `rentals:write`). The response carries `numero_contrat`, the **contract number**, which is the rental's key everywhere else in the API. The spec warns on this field: *"This field is not editable after creation. ALWAYS fill it with the correct value"* — get it right the first time; there is no rename.
3. **Build the layout.** For each room, in this order:
   - `bedrooms_create` (`POST /api/v3/bedrooms/`) — beds are nested inside the bedroom body.
   - `bathrooms_create` (`POST /api/v3/bathrooms/`).
   - `subrooms_create` (`POST /api/v3/subrooms/`) for anything that is neither, using `subroom_types_list` for the allowed types.
   All three take `rental_id` = the `numero_contrat` from step 2 and need `rentals:write`.
4. **Attach amenities** — `rental_amenities_create` (`POST /api/v3/rental_amenities/`, scope `amenities:write`). Pull the valid catalogue first with `amenities_list`; amenity ids are platform-wide, not per-property.
5. **Write the descriptions** — `descriptions_create` (`POST /api/v3/descriptions/`). One row per language; the model is multi-locale because the platform runs across UK, FR, PT, ES, CH and UAE markets.
6. **Upload photos** — `photos_create` (`POST /api/v3/photos/`, scope `photos:write`). Order matters for OTA listings; the hero image is the first.
7. **Define check-in steps** — `check_in_steps_create` (`POST /api/v3/check_in_steps/`) once per step, then `check_in_steps_order_create` (`POST /api/v3/check_in_steps/order/`) to set the sequence in one call. Do not try to order by re-creating.
8. **Create the OTA listing** — `listings_create` (`POST /api/v3/listings/`). Read `rental_platform_accounts_list` first to find which Airbnb / Booking.com / Vrbo account the listing should hang off.
9. **Go live** — `listings_enable_create` (`POST /api/v3/listings/{id}/enable/`) then `rentals_enable_create` (`POST /api/v3/rentals/{numero_contrat}/enable/`).

## Verify

`rentals_retrieve` (`GET /api/v3/rentals/{numero_contrat}/`) returns the assembled property with `listings`, `images`, `listing_photos`, `smart_locks` and `guest_control` inlined. Check `listings` is non-empty and the property status is active before you tell anyone it is onboarded.

## Reversing this

- **Reversible:** `rentals_disable_create` (`POST /api/v3/rentals/{numero_contrat}/disable/`) takes the property back off sale; `listings_disable_create` does the same per channel. Both are status flips, not deletes.
- **NOT reversible:** `rentals_destroy`, `bedrooms_destroy`, `bathrooms_destroy`, `photos_destroy` and every other `*_destroy` in this API are hard deletes with no restore operation and no documented retention. Prefer `disable` over `destroy` unless you are certain.

## Errors you will actually hit

`403 {"detail": "Authentication credentials were not provided."}` means the token expired — refresh at `/o/token/` and retry. Field validation comes back as `{"<field>": ["This field is required."]}`. See `errors/guestready-problem-types.yml`.
