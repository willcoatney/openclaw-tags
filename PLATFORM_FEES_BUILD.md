# Platform Fees Build — 2026-08-10

## Summary
Implemented the full 1.5% contractor platform fee system. Soft gate: the bid award always succeeds regardless of fee outcome. All 8 commits are local on `main` — not pushed.

---

## What Was Built

### 1. Migration — `supabase/migrations/20260810_platform_fees.sql`
Added to the `bids` table:
- `platform_fee_status TEXT NOT NULL DEFAULT 'not_applicable'` — values: `not_applicable | charged | pending | failed | waived`
- `platform_fee_failed_at TIMESTAMPTZ`
- `platform_fee_charged_at TIMESTAMPTZ`
- Index: `idx_bids_platform_fee_status`

Migration was applied to Supabase via management API. ✅

---

### 2. Email Templates — `src/lib/email-templates/platform-fees.ts`
Two HTML email template functions:
- `feeChargedEmail(projectTitle, feeAmount, cardLast4)` — confirmation with green fee card
- `feeDeclinedEmail(projectTitle, feeAmount, billingUrl)` — declined card notice with CTA button and 48h warning

Dark-themed HTML matching TAGS brand (deep navy, oklch color palette).

---

### 3. Core Helper — `src/lib/platform-fee.ts`
`chargePlatformFee(bid, project, routeContext)` — shared between award and retry routes.

**Flow:**
1. Look up contractor profile → org → `stripe_customer_id`
2. Fetch default card via `stripe.paymentMethods.list({ customer, type: 'card', limit: 1 })`
3. Calculate fee: `max($1.00, bid.amount × 1.5% × 100 cents)`
4. Create PaymentIntent with `off_session: true, confirm: true`
5. **On succeeded:** update bid → `status=charged, stripe_charge_id, platform_fee_cents, platform_fee_charged_at`. Send fee confirmation email.
6. **On failure (card declined / Stripe error):** update bid → `status=failed, platform_fee_failed_at`. Log error. Send declined SMS + email.
7. **On no customer or no card:** update bid → `status=pending`. Log. Send pending notification SMS + email.

Returns typed `FeeChargeResult` — never throws.

---

### 4. Award Route — `src/app/api/bids/[id]/award/route.ts`
Replaced the old invoice-based charge with `chargePlatformFee`. Key changes:
- Uses PaymentIntent instead of invoice items
- Tracks fee status on the bid record
- Winner SMS now appends: *"A $X platform fee was charged to your card ending XXXX"* on success
- Response includes `feeStatus` field
- Belt-and-suspenders try/catch around the helper call (award never fails due to fee)

---

### 5. Retry Route — `src/app/api/admin/retry-fee/route.ts`
`POST /api/admin/retry-fee`
- Admin auth required
- Accepts `{ bidId }`
- Verifies fee status is `failed` or `pending` before retrying
- Calls `chargePlatformFee` — returns result with new status

---

### 6. Waive Route — `src/app/api/admin/waive-fee/route.ts`
`POST /api/admin/waive-fee`
- Admin auth required
- Accepts `{ bidId }`
- Only allows waiving `failed` or `pending` fees
- Sets `platform_fee_status = 'waived'`

---

### 7. Data Route — `src/app/api/admin/platform-fees/route.ts`
`GET /api/admin/platform-fees?status=<filter>`
- Admin auth required
- Returns `{ rows, summary: { totalCollectedCents, pendingCount, failedCount } }`
- Joins bids → projects (title) and user_profiles (contractor name/email)
- Excludes `not_applicable` rows
- Summary always computed over all statuses (not filtered)

---

### 8. Admin Page — `src/app/dashboard/admin/platform-fees/page.tsx`
Client component with:
- **Summary cards:** Total Collected ($), Pending (#), Failed (#)
- **Filter tabs:** All | Charged | Pending | Failed
- **Table columns:** Project, Contractor (name + email), Bid Amount, Fee, Status badge, Date, Actions
- **Status badges:** green=charged, yellow=pending, red=failed, gray=waived
- **Retry button** on failed/pending rows → calls retry-fee route
- **Waive button** on failed/pending rows → confirms + calls waive-fee route
- **Auto-refresh** every 30 seconds (matches error-logs page pattern)
- Toast notifications on action success/failure

---

### 9. Nav Link — `src/components/DashboardNav.tsx`
Added `{ href: '/dashboard/admin/platform-fees', label: 'Platform Fees' }` to the admin nav between Contractors and Error Logs.

---

## Commits (8, all local on `main`)
```
9b2aaea feat: add Platform Fees link to admin nav
3bfc730 feat: add Platform Fees admin page with summary cards, table, retry/waive actions
394e14b feat: add GET /api/admin/platform-fees data route with summary aggregates
4441cf6 feat: add POST /api/admin/retry-fee route
a15b62e feat: add POST /api/admin/waive-fee route
41cd882 feat: wire chargePlatformFee into award route with fee status in winner SMS
8aff362 feat: add chargePlatformFee helper (1.5% PaymentIntent, soft gate, SMS/email)
e10968f feat: add feeChargedEmail and feeDeclinedEmail templates
72df536 chore: add platform_fee_status columns and index to bids table
```

## Build
`npm run build` — ✅ Compiled successfully, no type errors. Warnings are pre-existing `<img>` lint warnings across unrelated pages.

## Notes for Will
- The Stripe API version is `2026-07-29.dahlia` (already in `stripe.ts`)
- The `subscription_status` check from the old invoice flow was **removed** — platform fees apply to all awarded bids regardless of subscription status. If you want to gate on subscription, add that check back inside `chargePlatformFee`
- The billing URL for declined/pending emails points to `/dashboard/billing` — ensure that page has a card update flow
- The admin data query caps at 500 rows — add pagination if volume warrants it
