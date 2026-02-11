# Shared Hosting Runbook (Hostinger-Compatible)

## 1. Runtime Expectations
- PHP + MySQL + cron only
- No daemonized queue workers assumed
- Resource constraints expected

## 2. Laravel Runtime Configuration
- `QUEUE_CONNECTION=database`
- `CACHE_DRIVER=file` or DB-backed depending plan limits
- `SESSION_DRIVER=database`
- schedule runner via cron every minute

## 3. Required Cron Entries
- `* * * * * php /path/to/artisan schedule:run >> /dev/null 2>&1`
- Optional staggered heavy job command every 5 minutes for reporting snapshots

## 4. Job Design Rules
- Use chunked job batches (small chunk size)
- Max attempts with exponential backoff
- Dead-letter state for unresolved failures
- Protect all jobs with tenant boundaries

## 5. Performance Guardrails
- Precompute heavy reporting daily snapshots
- Index all date-range and external-id lookup tables
- Cache static config and routes on deploy
- Keep media optimized at upload

## 6. Incident Playbook
- Step 1: Check failed integration jobs
- Step 2: Requeue safe idempotent jobs only
- Step 3: Resolve conflict cases manually
- Step 4: Verify payment reconciliation before closing incident

## 7. Upgrade Thresholds (When Shared Hosting Is No Longer Enough)
Escalate to VPS/cloud when any sustained threshold is exceeded:
- >250 rooms per tenant
- >20k monthly bookings per tenant
- >100k integration events/day total
- Cron backlog not clearing within 10 minutes

## 8. Autonomous Operations (No Dedicated Maintenance Team)
Design and operate as self-serve by default:
- Auto-retry transient integration failures with bounded backoff
- Auto-close resolved conflicts when deterministic rules can be applied
- Nightly health-check job writes a human-readable summary in admin dashboard
- Built-in runbooks in admin UI for common failures (credentials expired, mapping missing, payment webhook mismatch)
- Safe-mode switches to pause specific connectors without breaking direct bookings

## 9. Minimum Self-Serve Monitoring Checklist
- Booking funnel health (search → payment → confirmation)
- OTA sync freshness (last successful pull/push timestamps)
- PMS sync freshness and backlog count
- Payment reconciliation mismatch count
- Cron execution heartbeat
