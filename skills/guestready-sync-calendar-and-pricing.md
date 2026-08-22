---
name: guestready-sync-calendar-and-pricing
description: Block and unblock availability, push daily rates in bulk, manage rate rules and pricing algorithms, and wire an external iCal feed into a RentalReady property.
api: RentalReady API
base_url: https://pms.rentalready.io/api/v3/
generated: '2026-08-22'
method: generated
source: openapi/guestready-rentalready-openapi.yml
scopes:
  - calendar:read
  - calendar:write
  - pricing:read
  - pricing:write
  - rentals:read
  - rentals:write
operations:
  - calendar_block_create
  - calendar_unblock_create
  - daily_prices_list
  - daily_prices_create
  - daily_prices_bulk_update_create
  - pricing_list
  - pricing_bulk_update_create
  - rate_rules_create
  - rate_rules_destroy
  - los_list
  - icals_list
  - icals_create
  - icals_partial_update
  - icals_destroy
  - rentals_block_partial_update
  - rentals_unblock_partial_update
---

# Sync calendar, rates and iCal feeds

## Two different kinds of "block"

RentalReady blocks availability at two levels and they are not interchangeable:

- **Date-range block** — `calendar_block_create` (`POST /api/v3/calendar/block/`) and its exact reversal `calendar_unblock_create` (`POST /api/v3/calendar/unblock/`). Scope `rentals:write`. Use this for a maintenance window or an owner stay.
- **Whole-property block** — `rentals_block_partial_update` (`PATCH /api/v3/rentals/{numero_contrat}/block/`) and `rentals_unblock_partial_update`. This takes the property out of circulation entirely.

Both are cleanly reversible. Neither has a documented time limit.

## Pushing rates

1. **Read what is there** — `daily_prices_list` (`GET /api/v3/daily_prices/`, scope `pricing:read`), filtered by `rental_id`.
2. **Write one day** — `daily_prices_create` (`POST /api/v3/daily_prices/`).
3. **Write many days** — `daily_prices_bulk_update_create` (`POST /api/v3/daily_prices/bulk_update/`). **Use this.** With a 400 request/minute ceiling and no rate-limit response header to back off against, a per-day loop over a season will exhaust the budget; the bulk endpoint is the only way to price a portfolio at scale.
4. **Pricing algorithms** — `pricing_list` / `pricing_create` / `pricing_bulk_update_create` (`POST /api/v3/pricing/bulk_update/`) manage the pricing model itself rather than individual nights.
5. **Rate rules** — `rate_rules_create` (`POST /api/v3/rate_rules/`) for recurring adjustments. `rate_rules_destroy` is a **hard delete with no restore** — read the rule first if you may need to recreate it.
6. **Length-of-stay restrictions** — `los_list` (`GET /api/v3/los/`).

## Connecting an external calendar (iCal)

RentalReady speaks **iCalendar (RFC 5545)** for calendar exchange with systems it has no native integration for. This is the lowest-friction way to keep a third-party channel in sync.

- `icals_list` (`GET /api/v3/icals/`, scope `rentals:read`) — existing syncs, with `last_sync`.
- `icals_create` (`POST /api/v3/icals/`, scope `rentals:write`) — body carries `rental_id` (the contract number), `platform_id` (from `reservation_platforms_list`), `ical_url` (*"The URL to fetch the iCal calendar from"*, max 200 chars), `import_summary_as_guest_name` and `is_active`.
- `icals_partial_update` — flip `is_active`; **only active syncs are processed**, per the spec's own field description.
- `icals_destroy` — hard delete, no restore.

Note the direction: `ical_url` is an **import** URL. This is RentalReady pulling a feed, not publishing one.

## Pagination

Every collection here is `limit`/`offset` paginated inside a `{count, next, previous, results, limit}` envelope. Follow `next` — it is an absolute URL — rather than incrementing `offset` yourself.
