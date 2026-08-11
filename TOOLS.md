# TOOLS.md — TAGS Stack & Dev Standards

## Tech Stack

- Frontend/Backend: Next.js 14 App Router + TypeScript (strict mode)
- Database: Supabase (Postgres + Auth + Storage + RLS)
- Hosting: Netlify (auto-deploy from GitHub main)
- AI: Anthropic Claude claude-sonnet-4-6 (SOW generation with vision)
- SMS: Twilio
- Email: Resend
- UI: Tailwind CSS + shadcn/ui

## Architecture Rules (non-negotiable)

- Every table has organization_id — multi-tenancy from day one, never retrofit
- RLS enabled on every table before any data goes in
- All server-side API routes use createAdminClient() (service role key) — never anon client for writes
- npm run build must pass before every commit — no exceptions
- TypeScript strict mode — no any types
- Never expose service role key client-side

## User Roles

- pm — Property Manager (posts projects, reviews bids)
- contractor — Contractor (views projects, submits bids)
- admin — TAGS internal (approves contractors, sees everything)

## RLS Pattern

- PMs see only their org's data
- Contractors see all 'open' projects + only their own bids
- Admin sees everything (server-side only, service role)

## AI SOW Generation

Model: claude-sonnet-4-6 with vision
Input: project_type + description (layman's terms) + photos
Output: structured SOW with 7 sections:

1. Project Overview
2. Scope of Work (numbered steps)
3. Materials Required
4. Labor Requirements
5. Site Safety Requirements
6. Cleanup & Disposal
7. Warranty Expectations

Rule: never include pricing in the SOW

## Key Paths (populate as project grows)

- GitHub repo: TBD
- Supabase project: TBD
- Netlify site ID: TBD
- Local workspace: /data/.openclaw/workspace/tags-app/
