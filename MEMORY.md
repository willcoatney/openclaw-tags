# MEMORY.md — Long-Term Memory

Last Updated: 2026-05-05

## Business Model (confirmed)

- No subscription fees — for anyone
- TAGS takes 1–5% of every project that runs through the platform
- PMs get a free, effortless tool
- Contractors get qualified, scoped leads — they're grateful to be involved
- Transaction fee introduced once volume is flowing
- Aligns incentives: TAGS only wins when contractors get paid

## Core Product Insight

The AI-generated Scope of Work IS the product. PM describes a problem in plain language, uploads a photo — AI returns a professional construction document a contractor can act on immediately. No other contractor marketplace does this. It's the moat.

## Team

- Mo (Maurice): Founder, idea originator, multifamily industry expert. The reason this exists.
- Chris: Business advisor (Will's father-in-law). Strategic, measured, experienced.
- Dale: Chris's business partner, friend of Mo's. Industry connections, go-to-market.
- Will: Tech lead. SaaS experience. Building everything.

## Tech Stack (decided)

Next.js 14 + TypeScript + Supabase + Netlify + Claude API + Twilio + Resend

## Git Standards
- Granular commits — one logical change per commit, not batched
- Conventional commit format: fix:, feat:, chore:, docs:
- Summary line + bullet points in body for anything non-trivial
- Makes git blame, reverts, and PR reviews clean

## Key Decisions

- Multi-tenant RLS architecture from day one (organization_id on every table)
- AI SOW uses claude-sonnet-4-6 with vision (photo + description → 7-section SOW)
- Stripe deferred — show pricing now, enforce billing in next sprint
- Build PM tool first, contractor experience second
- SMS notifications via Twilio for all key events

## Pricing Model (updated 2026-08-08)

- PMs: $50/month per account (per email address / organization) — confirmed by Mo 2026-08-08
- Contractors: 1.5% platform fee charged off-session when bid is awarded — soft gate policy (award goes through even if card declines, contractor gets 48h to resolve)
- Early access accounts grandfathered at current rates
- Stripe integration: shipped 2026-08-08 — awaiting Will to add env vars + run DB migrations

## Team Timezone
Central Time (CT) — all scheduling and reminders should use CT

## Infrastructure

- OpenClaw instance: openclaw-tags container on srv1522624.hstgr.cloud
- Telegram group: TAGS (chat ID: -1003871141582, upgraded from -5272805541 supergroup migration 2026-08-11)
- Bot: @Mr_Macro_bot (rename before showing partners)
- Anthropic API: TAGS org (separate from Will's personal account)
- Supabase project URL: https://sdjhnjuyfgwnjubzsfid.supabase.co
- Supabase anon key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNkamhuanV5Zmd3bmp1YnpzZmlkIiwicm9sZSI6ImFub24iLCJpYXQiOjE3Nzc5NTQ5MDAsImV4cCI6MjA5MzUzMDkwMH0.eygHJ5H6ICzcQw950eFzyLS_SQy3gECbCd8XmHUbKIs
- Supabase service role key: stored in tags-app/.env.local
- Supabase management API token: stored in tags-app/.env.local (key: SUPABASE_MANAGEMENT_TOKEN)
- GitHub repo: git@github.com:willcoatney/tags.git
- Deploy key (public): ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPU6X/PuiJzvqTrVREiZu6J9bNcEzaoML0HEMvgOdxUq atlas-tags-deploy
- Deploy key (private): /data/.openclaw/workspace/.ssh/tags_deploy
- Atlas workspace repo: git@github.com:willcoatney/openclaw-tags.git
- Workspace deploy key (public): ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOt790ZM4rqEU37hwINJsoOFzVNvtvfpwjsCCznyvhQq atlas-openclaw-workspace
- Workspace deploy key (private): /data/.openclaw/workspace/.ssh/openclaw_workspace_deploy
- Git push command: GIT_SSH_COMMAND="ssh -i /data/.openclaw/workspace/.ssh/openclaw_workspace_deploy -o StrictHostKeyChecking=no" git push

## Regional Manager Feature (shipped 2026-07-15)

- New role: `asset_manager` (Regional Manager in UI)
- New tables: portfolios, portfolio_organizations, portfolio_invites (migration run ✓)
- New routes: /register (role chooser), /register/rm, /join/portfolio/[token]
- New dashboards: /dashboard/asset-manager, /dashboard/asset-manager/projects, /dashboard/asset-manager/properties
- PM pricing updated to $50/community beta (invite-only) across landing page + register flow
- Portfolio invite flow: RM generates link → PM clicks → registers → auto-linked to portfolio
- Path B: PM can connect to portfolio from their dashboard via invite code

## Legal Monitor
- Monthly regulatory watch active as of 2026-08-08 (Dale's request)
- Watch file: LEGAL_MONITOR.md
- Runs 1st of each month via heartbeat
- Areas: contractor referral laws, TCPA/SMS, payment regs, state privacy laws, IC classification, contractor licensing liability
- Alert format: plain language summary + code/policy/legal counsel recommendation

## Netlify Access
- PAT: nfp_KEgY8xQQRabozjeiS1RhkhoA3HQFNJ4i1449
- Site ID: c9c31f90-0eae-403e-b229-1e10f239736d
- Account slug: willcoatney
- Site name: tags-web-app
- API pattern: POST https://api.netlify.com/api/v1/accounts/willcoatney/env?site_id=c9c31f90-0eae-403e-b229-1e10f239736d

## Stripe (Live)
- Secret key: in tags-app/.env.local
- PM price ID: price_1U2xuNBhkjXZj7azTOiyZGfL ($50/month)
- Contractor price ID: price_1U2xuNBhkjXZj7azBqh77Cbs ($100/year)

## Stripe (Test)
- Test key: in tags-app/.env.local
- PM price ID: price_1U2xvWAz0jKhTU5aPgQaXueb ($50/month)
- Contractor price ID: price_1U2xvXAz0jKhTU5aF3GLBvro ($100/year)

## Open Items

- [ ] Rename bot from @Mr_Macro_bot to something on-brand
- [ ] Add Mo, Chris, Dale Telegram IDs to allowedSenders config
- [x] Create GitHub repo for TAGS app — git@github.com:willcoatney/tags.git
- [x] Set up Supabase project for TAGS — https://sdjhnjuyfgwnjubzsfid.supabase.co
- [x] Run supabase/schema.sql in Supabase SQL editor
- [x] Build and deploy MVP — https://www.tagyourproject.com (primary domain live, SSL active)
- [x] Add tagyourproject.com domain to Netlify — live at https://www.tagyourproject.com
- [x] Set up Resend (email) — RESEND_API_KEY live in Netlify; tagyourproject.com domain added + DKIM/SPF verified. DMARC record optional/pending.
- [x] Set up Twilio (SMS) — fully live and tested. All 4 vars in Netlify + .env.local. Messages must route via TWILIO_MESSAGING_SERVICE_SID (MG4b057ef5...) for A2P compliance. Direct From-only sends fail.
- [x] Regional Manager role + portfolio dashboard — shipped and live
- [ ] Google account — Will creating Gmail for Atlas access (Gmail, Docs, Sheets)
