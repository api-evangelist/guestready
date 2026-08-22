---
name: guestready-handle-guest-messages-and-issues
description: Read the unified guest inbox, reply to a guest, flag a conversation, and turn a guest problem into a tracked issue and a dispatched field mission.
api: RentalReady API
base_url: https://pms.rentalready.io/api/v3/
generated: '2026-08-22'
method: generated
source: openapi/guestready-rentalready-openapi.yml
scopes:
  - conversations:read
  - conversations:write
  - messages:read
  - messages:write
  - inquiries:read
  - issues:read
  - issues:write
  - missions:read
  - missions:write
operations:
  - conversations_list
  - conversations_retrieve
  - conversations_create
  - conversations_set_flag_partial_update
  - conversation_flags_list
  - messages_list
  - messages_create
  - inquiries_list
  - issues_create
  - issues_list
  - issues_partial_update
  - issues_close_partial_update
  - issue_notes_create
  - issue_categories_list
  - missions_create
  - missions_update_agent_partial_update
  - missions_mark_mission_as_done_partial_update
  - missions_cancel_partial_update
  - agents_list
---

# Handle guest messages and turn them into work

## Read the inbox

1. `conversations_list` (`GET /api/v3/conversations/`, scope `conversations:read`) — the unified thread list across OTA channels and direct bookings. Filter with `created_after` / `modified_after` for incremental polling. **There are no webhooks in this API** — polling those change-window filters is the only event mechanism available.
2. `conversations_retrieve` (`GET /api/v3/conversations/{id}/`) for one thread.
3. `messages_list` (`GET /api/v3/messages/`, scope `messages:read`), filtered by `conversation_id`.
4. `inquiries_list` (`GET /api/v3/inquiries/`, scope `inquiries:read`) for pre-booking enquiries, which are a separate resource from conversations.

## Reply and triage

- **Reply** — `messages_create` (`POST /api/v3/messages/`, scope `messages:write`).
- **Flag a thread** — `conversations_set_flag_partial_update` (`PATCH /api/v3/conversations/{id}/set_flag/`, scope `conversations:write`). Body requires `flag_id`; valid ids come from `conversation_flags_list`. Omitting it returns `400 {"flag_id": ["This field is required."]}`, and lacking the permission returns `403 {"detail": "Guest messaging write not allowed, permission missing."}` — note that the scope alone is not sufficient, the user profile must also grant guest-messaging write.

## Escalate to an issue

1. `issue_categories_list` — pick the category first.
2. `issues_create` (`POST /api/v3/issues/`, scope `issues:write`).
3. `issue_notes_create` (`POST /api/v3/issue_notes/`) to add context as it develops.
4. `issues_close_partial_update` (`PATCH /api/v3/issues/{id}/close/`) when resolved.

Platform-raised **incidents** are a different resource (`incidents_list`). Their rules are one-way: the contract states *"An archived incident cannot change status"* — once archived, an incident cannot be closed or reopened. Close before you archive, never after.

## Dispatch a field mission

1. `agents_list` (`GET /api/v3/agents/`, scope `agents:read`) — the field agents available, scoped to an office.
2. `missions_create` (`POST /api/v3/missions/`, scope `missions:write`) — a cleaning, check-in or maintenance task against a rental.
3. `missions_update_agent_partial_update` (`PATCH /api/v3/missions/{id}/update_agent/`) to assign or reassign.
4. `missions_mark_mission_as_done_partial_update` (`PATCH /api/v3/missions/{id}/mark_mission_as_done/`) on completion.

**Reversal:** `missions_cancel_partial_update` (`PATCH /api/v3/missions/{id}/cancel/`) cancels a dispatched mission. It is a status transition, no window is documented, and it does not un-notify an agent who has already been told — check `mission_dashboards` state before assuming a cancellation is invisible to the field.
