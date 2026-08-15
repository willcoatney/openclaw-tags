# DAILY_REPORT.md

## Purpose

Run every morning at 8:00 AM CT. Surface red flags, operational misses, and competitive signals before they become problems. If everything is clean, log it silently and stay quiet.

## Schedule

Daily at 8:00 AM CT. Triggered via heartbeat.

## Report Sections

Run all checks. Only send a message if at least one flag is found. Group findings by severity.

---

### 🔴 SECTION 1 — Build & Deploy Health

Check Netlify last deploy for site ID: c9c31f90-0eae-403e-b229-1e10f239736d

```
curl -s -H "Authorization: Bearer nfp_KEgY8xQQRabozjeiS1RhkhoA3HQFNJ4i1449" \
  "https://api.netlify.com/api/v1/sites/c9c31f90-0eae-403e-b229-1e10f239736d/deploys?per_page=1" \
  | jq '.[0] | {state, error_message, created_at, branch}'
```

**Flag if:** state is not "ready". Include error_message in alert.
**Pass if:** state == "ready"

---

### 🔴 SECTION 2 — Stuck Projects (No Bids in 48h)

Query Supabase for open projects with no bids after 48 hours.

```
curl -s \
  -H "apikey: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNkamhuanV5Zmd3bmp1YnpzZmlkIiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImlhdCI6MTc3Nzk1NDkwMCwiZXhwIjoyMDkzNTMwOTAwfQ.ede0QSrJS7K6Oq-PJip6SgkNxLd9aYs0ChP_uLps4gM" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNkamhuanV5Zmd3bmp1YnpzZmlkIiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImlhdCI6MTc3Nzk1NDkwMCwiZXhwIjoyMDkzNTMwOTAwfQ.ede0QSrJS7K6Oq-PJip6SgkNxLd9aYs0ChP_uLps4gM" \
  "https://sdjhnjuyfgwnjubzsfid.supabase.co/rest/v1/projects?select=id,title,created_at,status&status=eq.open"
```

Cross-reference against bids table. Flag any project with status=open, created more than 48h ago, and zero bids.

**Flag if:** any such project exists. List project titles + age.
**Pass if:** no stuck projects.

---

### 🔴 SECTION 3 — Contractors Stuck in Pending Approval

```
curl -s \
  -H "apikey: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNkamhuanV5Zmd3bmp1YnpzZmlkIiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImlhdCI6MTc3Nzk1NDkwMCwiZXhwIjoyMDkzNTMwOTAwfQ.ede0QSrJS7K6Oq-PJip6SgkNxLd9aYs0ChP_uLps4gM" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNkamhuanV5Zmd3bmp1YnpzZmlkIiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImlhdCI6MTc3Nzk1NDkwMCwiZXhwIjoyMDkzNTMwOTAwfQ.ede0QSrJS7K6Oq-PJip6SgkNxLd9aYs0ChP_uLps4gM" \
  "https://sdjhnjuyfgwnjubzsfid.supabase.co/rest/v1/contractor_profiles?select=id,company_name,created_at&approval_status=eq.pending"
```

**Flag if:** any contractor has been pending > 24 hours. They applied, we haven't responded. That's a bad first impression.
**Pass if:** no pending contractors over 24h.

---

### 🟡 SECTION 4 — Competitive Intelligence

Run web searches for recent news on direct competitors and relevant market moves. Focus on the last 7 days.

Search queries to run:
1. `"Lessen" OR "Homee" OR "Tulu" multifamily maintenance marketplace 2026`
2. `multifamily property maintenance software funding 2026`
3. `contractor marketplace multifamily apartment 2026`
4. `property management maintenance AI scope of work 2026`

**Flag if:** any result shows a competitor launching a feature similar to our AI SOW, raising a significant funding round, or expanding into multifamily.
**Pass if:** nothing material in the last 7 days.

Do NOT flag general industry news that doesn't affect competitive position.

---

### 🟡 SECTION 5 — Product Feature Gaps

Review this standing list of known gaps. Flag if a gap has been open > 14 days and hasn't been discussed recently in memory.

| Gap | Priority | Notes |
|-----|----------|-------|
| Contractor payment processing (Stripe) | High | Currently off-platform — biggest liability for retention |
| Mobile-optimized contractor flow | High | Contractors work from phones, not laptops |
| Bid comparison view for PMs | Medium | PMs must manually compare bids today |
| Contractor ratings visible on bid card | Medium | PMs can't see reputation at a glance |
| Auto-expiry for bids after project close | Low | Manual cleanup currently required |
| Push/email notifications for PMs on new bids | Medium | SMS only right now |

**Flag if:** any High priority gap has had no activity (no code commits, no discussion) in the past 7 days.
**Pass if:** at least one high-priority gap is actively being worked.

---

## Destination

Send daily reports to the TAGS Telegram group chat (ID: -1003871141582), not to Will's direct message thread.

## Alert Format

When sending the daily report, use this format:

```
🏗 TAGS Daily Report — [Date]

🔴 RED FLAGS
[list any critical issues]

🟡 WATCH ITEMS
[list any warnings or competitive intel]

✅ CLEAN
[list checks that passed]
```

If zero red flags and zero watch items: do NOT send a message. Log "DAILY_REPORT_OK [date]" to memory and stay silent.

## Logging

After each run, append one line to memory/daily-report-log.md:
`[date] — [OK / N flags] — [brief summary if flags]`
