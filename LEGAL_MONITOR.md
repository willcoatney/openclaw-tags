# LEGAL_MONITOR.md — Regulatory Watch for TAGS

## Purpose
Monthly scan for legal and regulatory changes that could affect TAGS operations.
Triggered by heartbeat on the 1st of each month.

## Last Run
Never (initialized 2026-08-08)

## Watch Areas

### 1. Contractor Referral & Marketplace Laws
- New state laws classifying lead-gen or referral platforms as "contractor referral agencies"
- States known to be active: CA, TX, FL, AZ, IL, NY
- Key risk: platforms that earn fees on contractor jobs may need state-level registration
- Search terms: "contractor referral platform law", "home services marketplace regulation [STATE]"

### 2. TCPA & SMS Regulations
- FCC rule changes on automated text messages / A2P messaging
- "Prior express written consent" definition updates
- Opt-out and STOP compliance requirement changes
- Search terms: "TCPA 2026 update", "FCC SMS marketing rules", "A2P 10DLC regulation"

### 3. Payment & Money Transmission
- State money transmitter license requirements for marketplaces
- FinCEN guidance on payment facilitators
- Stripe regulatory changes that affect our fee model
- Search terms: "money transmitter marketplace platform 2026", "payment facilitator state license"

### 4. Data Privacy (CCPA / State Privacy Laws)
- New state privacy laws going into effect (TX, FL, OR, MT, etc. have active laws)
- CCPA enforcement updates
- Any new "right to know / delete" requirements for business data
- Search terms: "state data privacy law 2026", "CCPA enforcement update"

### 5. Independent Contractor Classification
- Federal or state rulemaking on gig/marketplace worker classification
- DOL guidance affecting two-sided marketplaces
- Search terms: "independent contractor classification 2026", "gig worker marketplace law"

### 6. Construction Licensing Enforcement
- States adding digital marketplace liability for unlicensed contractor referrals
- Insurance verification mandate changes
- Search terms: "contractor licensing marketplace liability 2026", "unlicensed contractor platform liability"

## Alert Threshold
Alert the group if ANY of the following is found:
- A new law passed or bill signed affecting any watch area above
- A regulatory enforcement action against a comparable platform (Angi, Thumbtack, HomeAdvisor)
- A pending bill with >50% passage probability in a state where we operate

## No-Alert Condition
If no material changes found in any watch area: log "LEGAL_MONITOR_OK [month]" to this file and stay silent.

## Action Protocol
When an alert is triggered:
1. Summarize the change in plain language
2. Assess: does it require a code change, a policy change, or legal counsel?
3. Give one clear recommendation
4. Tag Will if code is required, tag Chris if legal counsel is warranted
