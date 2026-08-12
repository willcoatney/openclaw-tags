# HEARTBEAT.md

## Purpose

Check for blockers and open decisions. Alert the team if something needs attention. Stay silent (HEARTBEAT_OK) if everything is clean.

## Checks

### 1. Open Decisions
Review recent memory files. If any decision has been tabled but not resolved, surface it once per day.

### 2. Build Status
Check Netlify last deploy status. Alert if failed.

### 3. Stuck Projects
Check Supabase for projects stuck in 'open' with no bids after 48h.

### 4. Monthly Legal & Regulatory Monitor
Run on the 1st of each month. Read LEGAL_MONITOR.md and search for changes in each watch area.
If material changes found: alert group with plain-language summary + recommendation.
If nothing material: log "LEGAL_MONITOR_OK [month]" to LEGAL_MONITOR.md and stay silent.

### 5. Daily Report (8:00 AM CT)
Read DAILY_REPORT.md and execute all checks defined there.
Covers: build health, stuck projects, pending contractor approvals, competitive intelligence, product feature gaps.
Alert format and silence rules are defined in DAILY_REPORT.md.
Do not duplicate alerts already sent by other heartbeat checks in the same cycle.

## Silence Condition

If nothing needs attention: reply exactly HEARTBEAT_OK

## Alert Rules

- Never alert between 11pm–7am CT
- Don't repeat the same alert more than once per 6 hours
- Legal monitor: alert once per finding, not once per check cycle
- Daily report: send once per day at 8am CT; silence if all checks pass
