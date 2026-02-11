# Integration Adapter Contracts (OTA + PMS + Payments)

## 1. OTA Adapter Interface
Each OTA implementation must provide:
- `pullBookings(sinceCursor)`
- `pushAvailability(changeset)`
- `pushRates(changeset)`
- `pullCancellations(sinceCursor)`
- `verifyWebhook(signature, payload)` (optional where supported)

### OTA Mapping Requirements
- External room type ↔ internal room type
- External rate plan ↔ internal rate plan
- Occupancy rules mapping
- Meal plan mapping

### OTA Idempotency
Idempotency key format:
`{channel}:{hotel}:{external_event_id}:{event_type}:{event_version}`

## 2. PMS Adapter Interface
Each PMS implementation must provide:
- `pullReservations(sinceCursor)`
- `pushReservation(reservationPayload)`
- `pullGuestProfiles(sinceCursor)`
- `pushInventory(changeset)`
- `testConnection()`

### PMS Ownership Policy
Configurable per hotel:
- PMS authoritative for inventory/reservation status
- Platform authoritative for direct booking with reconciliation push

## 3. Payment Provider Interface
Each payment adapter must provide:
- `createIntent(orderContext)`
- `authorize(intentRef)`
- `capture(intentRef, amount)`
- `refund(transactionRef, amount)`
- `verifyWebhook(signature, payload)`
- `fetchTransactionStatus(transactionRef)`

## 4. Conflict Resolution Contract
When conflicting updates are detected:
1. Persist both payloads
2. Evaluate precedence policy
3. Auto-resolve if deterministic
4. Otherwise open `conflict_case` for staff review
5. On manual override, force audit reason code

## 5. Observability Contract
Every adapter action emits:
- correlation_id
- source system
- entity type
- entity identifier
- status (success/retry/fail)
- latency_ms
- attempt number

## 6. Sync Cursor and Freshness Contract
- Every pull method returns both records and next cursor token.
- Cursors are stored per connector + entity type + hotel.
- If cursor invalidation occurs, adapter must support bounded backfill window replay.
- Admin UI shows freshness timestamp per connector/entity stream.

## 7. Autonomous Recovery Contract
- Adapter errors must classify as: transient, auth, mapping, validation, provider-outage.
- Each class maps to a guided repair action in admin UX.
- Safe-mode pause/resume must be available per connector.
- Recovery actions must be idempotent and audit logged.
