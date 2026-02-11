# Domain Model & Core Data Contracts

## 1. Tenant and Identity
- `hotels` (tenant root)
- `hotel_settings` (branding, locale, policies)
- `users`
- `roles`, `permissions`
- `user_hotel_memberships`

**Rules**
- Every business table includes `hotel_id` unless globally scoped.
- Access checks require membership + permission claim.

## 2. Booking Domain
- `room_types`, `rooms`
- `rate_plans`, `rate_calendars`, `restrictions`
- `inventory_daily`
- `bookings`, `booking_rooms`, `booking_addons`
- `cancellations`, `booking_events`

**Rules**
- Booking states: initiated, pending_payment, confirmed, cancelled, no_show, checked_in, checked_out.
- Pricing snapshot copied onto booking at checkout to avoid mutable totals.

## 3. Guest Domain
- `guests`
- `guest_profiles`
- `guest_preferences`
- `loyalty_accounts`, `loyalty_ledger`
- `concierge_requests`

## 4. Payment Domain
- `payment_intents`
- `payment_transactions`
- `refunds`
- `invoices`
- `settlement_reports`

**Rules**
- No raw PAN/card storage.
- External transaction IDs unique per provider.

## 5. Integration Domain
- `channel_connections`
- `channel_mappings`
- `pms_connections`
- `pms_mappings`
- `integration_events`
- `integration_jobs`
- `integration_failures`
- `conflict_cases`

**Rules**
- All inbound payloads are persisted with checksum + idempotency key.
- Retries capped with dead-letter state.

## 6. SaaS Domain
- `subscription_plans`
- `hotel_subscriptions`
- `feature_flags`
- `usage_meters`
- `billing_invoices`

## 7. Audit and Compliance Domain
- `audit_logs`
- `consent_records`
- `data_export_requests`
- `data_deletion_requests`
- `retention_policies`

## 8. Relationship Integrity Rules
- `bookings.hotel_id` must equal all related line items (`booking_rooms`, `booking_addons`, `booking_guests`).
- `booking_rooms` must carry check-in/check-out and resolved `rate_plan_id` snapshot to prevent downstream pricing drift.
- `inventory_daily.available_count = total_count - blocked_count - sold_count + release_adjustments` (derived or materialized consistently).
- `integration_events` must always reference source connector and payload checksum.
- `conflict_cases` must link to conflicting event IDs and chosen resolution event.

## 9. Default Foreign-Key & Index Blueprint
- FK: `booking_rooms.booking_id -> bookings.id`
- FK: `booking_rooms.room_type_id -> room_types.id`
- FK: `inventory_daily.room_type_id -> room_types.id`
- FK: `payment_transactions.payment_intent_id -> payment_intents.id`
- FK: `channel_mappings.channel_connection_id -> channel_connections.id`
- FK: `pms_mappings.pms_connection_id -> pms_connections.id`
- Index: (`hotel_id`, `status`, `check_in`) on `bookings`
- Index: (`hotel_id`, `date`) on `inventory_daily`
- Index: (`provider`, `external_transaction_id`) unique on `payment_transactions`
