# Luxury Hotel Platform — Architecture & Delivery Plan (Laravel + MySQL, Shared Hosting Compatible)

## 1) Product Positioning & Strategic Intent

This product should be positioned as a **Luxury Hotel Commerce + Operations Platform** (not a basic website builder).

**Core value proposition:**
- Increase direct bookings and ADR (average daily rate).
- Preserve parity and visibility across OTA channels.
- Bridge existing PMS investments (Opera/eZee) without forcing replacement.
- Deliver branded, premium guest journeys that luxury properties expect.
- Package the whole stack into a resellable multi-tenant SaaS or deployable white-label edition.

---

## 2) Non-Negotiable Constraints (Hosting & Runtime)

### Shared-hosting-first constraints
- PHP/Laravel + MySQL only.
- Cron is available; long-running workers are not guaranteed.
- No Node runtime dependency.
- No container-only assumptions.
- Limited CPU/RAM and stricter process limits.

### Architectural consequences
- Queue driver: **database**.
- Background orchestration: cron-triggered `php artisan schedule:run` every minute.
- Integration sync: polling + webhook fallback where supported.
- Real-time UX features must degrade to polling/refresh.
- Heavy operations split into chunked, resumable jobs.

---

## 3) Proposed Product Architecture (Bounded Contexts)

Use a modular monolith inside Laravel (domain modules, strict boundaries, shared platform kernel).

1. **Core Identity & Tenancy**
   - Hotels, brands, users, roles, permissions, plan limits.
2. **Web CMS + Brand Engine**
   - Luxury themes, editable content blocks, localization, media.
3. **Booking Engine**
   - Inventory, rates, restrictions, offers, add-ons, cart/checkout.
4. **Payments Hub**
   - Stripe/Paytm/Razorpay adapters, capture/refund/reconcile.
5. **Guest & Loyalty**
   - Guest profile, preferences, communication consent, loyalty ledger.
6. **Operations Console**
   - Reservation desk, check-in/out support states, housekeeping board.
7. **Channel Manager (OTA)**
   - Booking.com/MakeMyTrip first, Expedia/Agoda extension points.
8. **PMS Bridge Layer**
   - Opera/eZee connector abstractions, mapping, sync policies.
9. **Reporting & Revenue Insights**
   - Occupancy, RevPAR, channel mix, payment settlement views.
10. **SaaS Control Plane**
   - Subscription billing, feature flags, white-label, license controls.
11. **Audit/Compliance Kernel**
   - Audit events, retention controls, PII masking, export/delete flows.

---

## 4) High-Level Build Sequence (Dependency Graph First)

### Layer 0 — Platform Foundation
- Tenancy model, RBAC, config framework, audit scaffolding, queue/scheduler conventions.

### Layer 1 — Revenue Core
- Room types, rate plans, restrictions, booking lifecycle, invoices, base analytics.

### Layer 2 — Money Movement
- Payment hub + reconciliation, partial deposits, pay-at-hotel logic.

### Layer 3 — External Distribution
- OTA channel manager core + mappings + sync jobs + conflict workflows.

### Layer 4 — PMS Bridges
- PMS abstraction + Opera/eZee connectors + master-data sync policies.

### Layer 5 — Luxury Differentiation
- Concierge requests, VIP flags, personalization offers, branded journey polish.

### Layer 6 — Productization & Scale-Out
- SaaS billing, white-label controls, tenant isolation hardening, onboarding automation.

---

## 5) Detailed Module Blueprint

## 5.1 Website + Booking Engine

### Capabilities
- Multi-language SEO pages and schema-ready metadata.
- Luxury media-heavy room/amenity/event pages.
- Availability search + pricing calendar.
- Dynamic pricing rules (seasonality, LOS, occupancy triggers).
- Upsells (spa, transfers, meals, experiences).

### Key design choices
- Keep CMS block-driven but constrained with curated components (avoid arbitrary page-builder complexity).
- Cache page fragments and search responses aggressively.
- Split reservation states: `initiated`, `pending_payment`, `confirmed`, `cancelled`, `no_show`, `checked_in`, `checked_out`.

## 5.2 OTA Channel Manager Layer

### MVP channels
- Booking.com
- MakeMyTrip

### API patterns
- Pull bookings periodically (idempotent import).
- Push inventory/rate changes through change-queue and retry policy.
- Capture webhooks where available but treat them as hints, not sole source.

### Mandatory internals
- Channel mapping table: room type, rate plan, occupancy, meal plan mapping.
- Versioned sync logs with correlation IDs.
- Conflict queue requiring human review when confidence is low.

## 5.3 PMS Integration Layer (Opera + eZee)

### Integration model
- Build a **PMS Adapter Contract** first.
- Implement `OperaAdapter`, `EZeeAdapter` against the same contract.
- Use canonical internal models and deterministic field mapping.

### Sync domains
- Reservations
- Guest profiles
- Inventory/availability (if PMS is master)
- Folio/invoice status where available

### Mastering policy
- If PMS connected and set as authoritative: PMS wins for inventory + reservation status.
- OTA/new direct bookings enter pending queue and reconcile into PMS.
- Admin override always writes explicit audit event.

## 5.4 Payments Infrastructure

### Payment orchestration
- `PaymentIntent` internal model abstracts provider specifics.
- Per-provider adapter for authorize/capture/refund/status check.
- Webhook verifier per gateway with replay protection.

### Flows
- Full prepay
- Partial deposit
- Pay-at-hotel with optional card guarantee token
- Corporate billing (invoice terms + ledger)

### Reconciliation
- Daily settlement reconciliation job.
- Mismatch dashboard + downloadable exception reports.

## 5.5 Admin Operations Console

### Front desk features
- Arrivals/departures board
- Reservation edits with policy checks
- Guest notes + preference tags

### Housekeeping
- Room status board (`dirty`, `cleaning`, `clean`, `out_of_order`)
- Quick assign actions and timestamped transitions

### Revenue management
- Rate overrides by date range
- Occupancy heatmaps
- Channel contribution insights

## 5.6 Guest Portal

- Booking history and modification requests.
- Invoice downloads and payment status.
- Preference capture (pillow type, dietary preferences, arrival details).
- Concierge request intake + status.
- Loyalty wallet (points earn/redeem policies configurable per tenant).

## 5.7 SaaS / Resale Layer

### Tenant model
- Single database, tenant_id scoped tables (MVP), with migration path to per-tenant DB for enterprise tier.

### Commercial controls
- Plan-based feature flags.
- Branded domains + white-label assets.
- Hotel onboarding wizard + checklist.
- License enforcement and usage metrics (rooms/bookings/API calls by plan).

---

## 6) Data Architecture & Core Tables (Conceptual)

### Core business entities
- `hotels`, `hotel_settings`, `brands`
- `users`, `roles`, `permissions`, `user_hotel_memberships`
- `room_types`, `rooms`, `rate_plans`, `rate_calendars`, `inventory`
- `bookings`, `booking_rooms`, `booking_guests`, `booking_addons`
- `guests`, `guest_preferences`, `guest_loyalty_ledgers`
- `payments`, `payment_transactions`, `refunds`, `invoices`

### Integration entities
- `channels`, `channel_connections`, `channel_mappings`
- `pms_connections`, `pms_mappings`
- `integration_jobs`, `integration_events`, `integration_failures`

### Governance entities
- `audit_logs`, `data_export_requests`, `data_deletion_requests`
- `feature_flags`, `subscription_plans`, `hotel_subscriptions`

---

## 7) Integration Reliability & Conflict Strategy

### Rules
1. Deterministic idempotency key per external event.
2. All inbound/outbound integration writes are event-logged.
3. Retries are exponential with max attempt + dead-letter state.
4. Manual conflict resolution UI for ambiguous updates.

### Conflict resolution policy
- Preferred precedence: PMS (if configured as master) > OTA confirmed booking > Direct booking edits.
- Last-update-wins only when source confidence is equal.
- Any forced override requires staff reason code and is auditable.

---

## 8) Performance Strategy (Shared Hosting Realities)

- Strict database indexing on date-range, hotel_id, status, external_id columns.
- Cache layers: config cache, route cache, query result cache, fragment cache.
- Heavy reports use pre-aggregated daily snapshots.
- API sync jobs use chunking and backpressure.
- Media optimization pipeline (compress, responsive variants) at upload time.

---

## 9) Security, Privacy, and Compliance Baseline

- No raw card storage; tokenization through gateway only.
- Encrypted secrets and connection credentials.
- RBAC with granular staff permissions (front desk, finance, admin, owner).
- Full audit trail for reservation/payment/status changes.
- GDPR-readiness: export/delete workflows, consent tracking, retention policies.
- Tenant data isolation controls and permission scoping tests.

---

## 10) Major Gaps Identified (Critical Before Build)

1. **Contractual/API access gaps**
   - Opera and some OTA APIs often require certification/commercial agreements.
2. **Channel certification effort not estimated**
   - Certification and sandbox testing can be longer than pure coding effort.
3. **Tax/regulatory model incomplete**
   - Need country/state tax engine strategy (VAT/GST/city tax/service charge).
4. **Rate parity and package complexity unspecified**
   - Need business rules for inclusions, add-ons, and parity safeguards.
5. **Cancellation/no-show policy matrix missing**
   - Policy variations per channel/season/rate plan are essential.
6. **Overbooking strategy undefined**
   - Need clear tolerance, alerting, and auto-reaccommodation workflow.
7. **Identity/security operations missing**
   - 2FA/SAML/SSO requirements for enterprise luxury chains not defined.
8. **Data residency/legal requirements not mapped**
   - Required for multi-country enterprise deals.
9. **Support/ops model absent**
   - Need L1/L2/L3 support model, uptime SLA, incident runbooks.
10. **Migration/onboarding tooling missing**
   - Existing booking/PMS import process is required for faster sales cycles.
11. **BI/export requirements unclear**
   - Luxury groups often demand custom exports to finance and BI systems.
12. **Shared-hosting scalability ceiling not quantified**
   - Define thresholds and upgrade path to VPS/cloud for larger properties.

---

## 11) Risk Register (with Mitigation)

- **R1: API certification delays** → start partner onboarding in parallel with core build.
- **R2: Shared hosting job bottlenecks** → enforce job budgets, chunk size caps, and cron staggering.
- **R3: Data sync conflicts** → launch with explicit conflict queue UI + retry controls.
- **R4: Payment dispute complexity** → design reconciliation first-class, not as report-only add-on.
- **R5: Multi-tenant security risk** → mandatory tenant-scoped query guards + security tests.

---

## 12) Implementation Roadmap by Capability Gates (Not Timeline-Based)

### Gate A — Sellable Core
- Luxury website + booking engine + Stripe/Paytm + invoices + basic admin.
- Outcome: direct-booking revenue product is demo/sellable.

### Gate B — Distribution Strength
- Booking.com + MakeMyTrip sync, mapping UI, conflict dashboard.
- Outcome: channel parity and operations confidence.

### Gate C — Enterprise Integrations
- Opera + eZee adapters, mastering policy controls, reconciliation tools.
- Outcome: fits hotels with existing PMS ecosystems.

### Gate D — Luxury Differentiators
- Guest portal, loyalty, concierge, personalization offers.
- Outcome: premium guest experience and repeat-stay value.

### Gate E — SaaS Productization
- Tenant plans, billing, white-label, onboarding automation.
- Outcome: reseller-ready repeatable product business.

---

## 13) Technical Delivery Standards (So Build Is Predictable)

- API-first internal module boundaries.
- Strict migration discipline with backward compatibility where possible.
- Idempotent job design everywhere integrations touch.
- Feature flags for all partner integrations.
- Observability baseline: structured logs, integration audit trails, alertable failure states.
- Test pyramid emphasis:
  - Unit tests for pricing/policy engines.
  - Integration tests for adapter contracts.
  - End-to-end smoke tests for booking/payment critical path.

---

## 14) Recommended Team Execution Model

- **Track 1:** Revenue Core (booking + payments).
- **Track 2:** Integrations (OTA + PMS contracts and adapters).
- **Track 3:** SaaS productization (tenancy, plans, white-label).
- **Track 4:** Luxury UX and guest experience.

Parallel tracks converge through shared platform contracts to reduce rework.

---

## 15) Definition of “Ready to Sell to Luxury Hotels”

A release is commercial-ready when all below are true:
- Direct booking funnel is stable and conversion-optimized.
- At least two OTA channels are production-proven.
- At least one PMS adapter is production-proven; second is near-ready.
- Payment reconciliation and refund workflows are audit-safe.
- Brand/theming quality meets premium expectation.
- Tenant onboarding can be completed by non-engineers using guided setup.
- Security/audit/privacy baseline is documented and test-validated.

---

## 16) Immediate Next Actions

1. Freeze domain glossary and source-of-truth ownership matrix.
2. Draft adapter contracts for OTA and PMS before implementation.
3. Finalize gate-wise acceptance criteria and demo scripts per gate.
4. Start API access/certification processes with Opera + OTA partners now.
5. Build a “pilot hotel onboarding kit” (data import templates, setup playbook, QA checklist).

This planning approach ensures the platform is engineered for **integration resilience, commercial reuse, and luxury-grade operations**, while staying compatible with shared hosting constraints from day one.
