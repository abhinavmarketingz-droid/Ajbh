# Capability Gates and Acceptance Criteria

## Gate A — Sellable Core
### Scope
- Multi-language luxury site pages
- Search availability and booking flow
- Rate plans and seasonal pricing
- Stripe + Paytm payment flows
- Invoice generation
- Reservation dashboard

### Acceptance
- End-to-end booking completed in < 2 minutes by UAT operator
- Deposit and full prepay both successful
- Refund recorded and visible in reconciliation report

## Gate B — OTA Distribution Strength
### Scope
- Booking.com adapter
- MakeMyTrip adapter
- Channel mapping interface
- Availability/rate sync jobs
- Conflict queue UI

### Acceptance
- Imported OTA booking reflected in reservation board
- Inventory push retry logic passes failure simulation
- Unresolved conflicts appear in review queue

## Gate C — PMS Enterprise Integration
### Scope
- PMS adapter contract implementation
- Opera adapter
- eZee adapter
- Mastering policy controls

### Acceptance
- Reservation sync round-trip completed for pilot property
- PMS authoritative mode blocks conflicting direct overrides unless admin confirmed

## Gate D — Luxury Experience Layer
### Scope
- Guest portal
- Concierge requests
- Preference profiles
- Loyalty ledger
- Personalized offer rules

### Acceptance
- Guest can submit request, track status, and download invoices
- Preference data influences targeted offer eligibility

## Gate E — SaaS Commercialization
### Scope
- Subscription plans
- Feature flags
- Usage metering
- White-label assets
- Onboarding checklist and setup wizard

### Acceptance
- New tenant can onboard without engineering access
- Plan limits enforced with visible metering
- White-label domain and branding applied successfully

## Gate F — Autonomous Operations Readiness
### Scope
- Connector health dashboard
- Guided repair flows for auth/mapping/failure classes
- Safe-mode controls per connector
- Automated daily operational digest

### Acceptance
- Non-technical admin can resolve expired credentials without engineering support
- Mapping issue can be identified and repaired through guided flow
- Connector safe mode can be toggled without blocking direct booking
- Health digest generated daily with actionable status summary
