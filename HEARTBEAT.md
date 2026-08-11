# HEARTBEAT.md

## Purpose

Check for blockers and open decisions. Alert the team if something needs attention. Stay silent (HEARTBEAT_OK) if everything is clean.

## Checks

### 1. Open Decisions
Review recent memory files. If any decision has been tabled but not resolved, surface it once per day.

### 2. Build Status (once app is deployed)
Check Netlify last deploy status. Alert if failed.

### 3. New Leads/Projects (once app is live)
Check Supabase for projects stuck in 'open' with no bids after 48h.

### 4. Monthly Legal & Regulatory Monitor
Run on the 1st of each month. Read LEGAL_MONITOR.md and search for changes in each watch area.
If material changes found: alert group with plain-language summary + recommendation.
If nothing material: log "LEGAL_MONITOR_OK [month]" to LEGAL_MONITOR.md and stay silent.

## Silence Condition

If nothing needs attention: reply exactly HEARTBEAT_OK

## Alert Rules

- Never alert between 11pm–7am CT
- Don't repeat the same alert more than once per 6 hours
- Legal monitor: alert once per finding, not once per check cycle
