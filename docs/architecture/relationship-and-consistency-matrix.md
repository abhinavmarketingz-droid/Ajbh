# Relationship & Consistency Matrix (Cross-File Validation)

This document is the canonical cross-check layer to prevent architectural conflicts across planning docs.

## 1) Source-of-Truth Ownership Matrix

| Domain Object | Primary Source | Secondary Source | Reconciliation Rule |
|---|---|---|---|
| Inventory availability | PMS (if authoritative mode ON), otherwise Platform | OTA snapshots/webhooks | Authoritative source wins; otherwise precedence + timestamp + confidence |
| Reservation status | PMS (if connected as master) | OTA / Platform | PMS > OTA confirmed > direct edit, with auditable override |
| Guest profile core fields | PMS (if mapped) | Platform guest profile | Merge non-conflicting attributes, manual review for identity mismatch |
| Payment settlement | Payment gateway settlement APIs | Platform transaction ledger | Gateway amount/status is final; platform tracks operational state |
| Room/rate mapping | Platform mapping UI | OTA/PMS catalogs | Platform mapping controls outbound pushes and import interpretation |

## 2) Entity Relationship Contract (Must Hold)

- `hotels (1) -> (N) users` via `user_hotel_memberships`.
- `hotels (1) -> (N) room_types`.
- `room_types (1) -> (N) inventory_daily` by date.
- `bookings (1) -> (N) booking_rooms` (multi-room and mixed room-type supported).
- `bookings (1) -> (N) payment_intents -> (N) payment_transactions`.
- `bookings (1) -> (N) booking_events` as immutable event history.
- `channel_connections (1) -> (N) channel_mappings`.
- `pms_connections (1) -> (N) pms_mappings`.
- `integration_events (1) -> (N) integration_jobs` execution attempts.
- `conflict_cases` always reference two or more competing updates.

## 3) Mandatory Referential and Uniqueness Rules

- All tenant-scoped business tables include `hotel_id`.
- `inventory_daily` unique key: (`hotel_id`, `room_type_id`, `date`).
- `booking_rooms` unique per (`booking_id`, `room_type_id`, `check_in`, `check_out`, `rate_plan_id`) at line-item granularity.
- `payment_transactions` unique per (`provider`, `external_transaction_id`).
- `integration_events` unique idempotency key per external event.
- Soft-delete policy must not break uniqueness (use partial unique indexes or archival strategy).

## 4) Conflict Resolution Matrix

| Conflict Type | Auto Resolve | Manual Queue | Notes |
|---|---|---|---|
| Same source duplicate event | Yes | No | Use idempotency key |
| PMS vs OTA status mismatch | Sometimes | Yes | Follow precedence, queue if confidence tie |
| OTA mapping missing | No | Yes | Block push; raise guided repair |
| Payment webhook status regression | No | Yes | Validate against provider fetchTransactionStatus |
| Guest identity mismatch | No | Yes | Prevent accidental profile merge |

## 5) Zero-Maintenance Operational Invariants

- No routine daily operation should require engineering or vendor intervention.
- Every critical failure path has a guided self-serve repair action in admin UI.
- Connector-specific safe mode must preserve direct-booking availability.
- Daily health digest generation is mandatory.
- Backlog/freshness metrics are visible without log access.

## 6) Cross-Document Consistency Checklist

Verify all related docs stay aligned on:
1. Polling-first sync semantics with webhooks as acceleration hints.
2. PMS/OTA/direct precedence policy and override audit requirement.
3. Multi-room `booking_rooms` support.
4. Inventory day-grain model and unique key contract.
5. Zero-maintenance-by-default operating model.
6. Shared-hosting limits and cron/database-queue strategy.
