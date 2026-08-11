# TAGS Silent Failure Audit
**Generated:** Atlas — Pre-Launch Security & Reliability Audit  
**Scope:** All files in `/tags-app/src/` — API routes, components, lib/, middleware  
**Status:** 33 issues found across 8 categories  

---

## Severity Summary

| Severity | Count |
|---|---|
| 🔴 CRITICAL | 8 |
| 🟠 HIGH | 10 |
| 🟡 MEDIUM | 8 |
| 🔵 LOW | 7 |

---

## 🔴 CRITICAL

---

### C-01 — Photo Upload: DB Insert Never Checked — Photos Are Orphaned
**File:** `src/app/api/upload/project-photo/route.ts`  
**Lines:** ~45-52  
**Category:** Silent Supabase failure + Missing null check  

**Current code:**
```ts
const { data: { publicUrl } } = admin.storage
  .from('project-photos')
  .getPublicUrl(storagePath)

await admin.from('project_photos').insert({   // ← NO error check
  project_id: projectId,
  storage_path: storagePath,
  public_url: publicUrl,
  uploaded_by: user.id,
  photo_type: photoType,
})

return NextResponse.json({ publicUrl, storagePath })  // ← Returns success regardless
```

**Problem:** If the `project_photos` INSERT fails (DB constraint, RLS, schema mismatch), the photo is stored in Supabase Storage forever but never recorded in the database. The API returns `200 OK`. The project will display no photos. There is no way to recover this without manual Storage cleanup. This is a silent data loss bug that will happen in production and be impossible to diagnose.

**Fix:**
```ts
const { error: insertError } = await admin.from('project_photos').insert({
  project_id: projectId,
  storage_path: storagePath,
  public_url: publicUrl,
  uploaded_by: user.id,
  photo_type: photoType,
})

if (insertError) {
  // Roll back — delete the orphaned file from storage
  await admin.storage.from('project-photos').remove([storagePath])
  return NextResponse.json({ error: 'Failed to record photo: ' + insertError.message }, { status: 500 })
}
```

---

### C-02 — Award Bid: Three Unguarded DB Writes Create Race Condition
**File:** `src/app/api/bids/[id]/award/route.ts`  
**Lines:** ~52-67  
**Category:** Silent Supabase failures + Race condition  

**Current code:**
```ts
// Award winning bid
await admin.from('bids').update({ status: 'awarded', ... }).eq('id', params.id)

// Reject all other bids
if (losingBids?.length) {
  await admin.from('bids').update({ status: 'rejected', ... }).in('id', ...)
}

// Update project status
await admin.from('projects').update({ status: 'awarded', ... }).eq('id', winningBid.project_id)
```

**Problem:** None of these three writes check for errors. If the project update fails, the winning bid is marked `awarded`, all other bids are `rejected`, but the project is still `open`. Contractors can re-bid on an already-awarded job. If the winning bid update fails, the project shows as `awarded` with no awarded bid. Supabase does not support cross-table transactions via the REST API — these must each be checked.

**Fix:**
```ts
const { error: awardErr } = await admin.from('bids')
  .update({ status: 'awarded', updated_at: new Date().toISOString() })
  .eq('id', params.id)
if (awardErr) return NextResponse.json({ error: 'Failed to award bid: ' + awardErr.message }, { status: 500 })

if (losingBids?.length) {
  const { error: rejectErr } = await admin.from('bids')
    .update({ status: 'rejected', updated_at: new Date().toISOString() })
    .in('id', losingBids.map(b => b.id))
  if (rejectErr) console.error('[award] Failed to reject losing bids:', rejectErr)
  // Non-fatal — log but don't block
}

const { error: projectErr } = await admin.from('projects')
  .update({ status: 'awarded', updated_at: new Date().toISOString() })
  .eq('id', winningBid.project_id)
if (projectErr) return NextResponse.json({ error: 'Failed to update project status: ' + projectErr.message }, { status: 500 })
```

---

### C-03 — Award Bid: SMS/Email Notifications Can Crash Response After Writes Are Complete
**File:** `src/app/api/bids/[id]/award/route.ts`  
**Lines:** ~130-148  
**Category:** SMS/Email fire-and-forget without error isolation  

**Current code:**
```ts
// Notify winning contractor
if (winner?.phone) {
  await sendSMS(winner.phone, `You've been awarded...`)  // ← throws on Twilio failure
}
if (winner?.email) {
  const { subject, html } = awardedWinnerEmail(...)
  await sendEmail(winner.email, subject, html)  // ← sendEmail silently catches, OK
}

// Notify losing contractors
for (const loser of (losingBids || [])) {
  const { data: loserProfile } = await admin.from('user_profiles')...
  if (loserProfile?.phone) {
    await sendSMS(loserProfile.phone, `...`)  // ← throws on Twilio failure, inside loop
  }
}
```

**Problem:** `sendSMS` throws on Twilio failure. At this point in the code, all DB writes are complete — the bid IS awarded. But the unhandled throw means the route returns 500, the client shows an error, and the PM may re-attempt the award (which will fail on the second try with "already awarded" confusion). The loser notification loop: a single SMS failure for loser #1 breaks the entire loop — no other losers are notified.

**Fix:**
```ts
if (winner?.phone) {
  try { await sendSMS(winner.phone, `...`) } catch (e) { console.error('[award] Winner SMS failed:', e) }
}
if (winner?.email) { ... }  // sendEmail already catches internally

for (const loser of (losingBids || [])) {
  const { data: loserProfile } = ...
  if (loserProfile?.phone) {
    try { await sendSMS(loserProfile.phone, `...`) } catch (e) { console.error('[award] Loser SMS failed:', e) }
  }
}
```

---

### C-04 — Publish Project: Status Update Has No Error Check
**File:** `src/app/api/projects/[id]/publish/route.ts`  
**Lines:** ~35-37  
**Category:** Silent Supabase failure  

**Current code:**
```ts
// Publish project
await admin.from('projects').update({ status: 'open', updated_at: new Date().toISOString() }).eq('id', params.id)

// Find matching contractors...
// SMS notifications go out...
return NextResponse.json({ success: true })
```

**Problem:** If the `projects` update fails, the project stays in `draft` status but contractors are already notified via SMS. The API returns `{ success: true }`. Contractors receive a link to a project they can't bid on. The PM dashboard shows the project as still unpublished (no apparent change), causing confusion and duplicate publish attempts.

**Fix:**
```ts
const { error: publishErr } = await admin.from('projects')
  .update({ status: 'open', updated_at: new Date().toISOString() })
  .eq('id', params.id)

if (publishErr) return NextResponse.json({ error: 'Failed to publish project: ' + publishErr.message }, { status: 500 })
```

---

### C-05 — Contractor Registration: Orphaned Auth Users on Profile Insert Failure
**File:** `src/app/api/auth/register/contractor/route.ts`  
**Lines:** ~30-50  
**Category:** Race condition + Missing error handling  

**Current code:**
```ts
const { data: org, error: orgErr } = await admin.from('organizations').insert(...).select().single()
if (orgErr) return NextResponse.json({ error: orgErr.message }, { status: 500 })

const { data: authData, error: authErr } = await admin.auth.admin.createUser(...)
if (authErr) return NextResponse.json({ error: authErr.message }, { status: 500 })

await admin.from('user_profiles').insert({      // ← NO error check
  id: authData.user.id, organization_id: org.id, role: 'contractor', ...
})

await admin.from('contractor_profiles').insert({  // ← NO error check
  user_id: authData.user.id, ...
})
```

**Problem:** The auth user is created before the profile is inserted. If `user_profiles` insert fails, the auth user exists in Supabase Auth with no profile record. The user can attempt to log in but the app has no profile to load — they're stuck in a broken state permanently. Same for `contractor_profiles`. The registration returns 200 (no error returned because it's not checked) but the account is broken.

**Fix:**
```ts
const { error: profileErr } = await admin.from('user_profiles').insert({ ... })
if (profileErr) {
  // Rollback: delete orphaned auth user
  await admin.auth.admin.deleteUser(authData.user.id)
  return NextResponse.json({ error: 'Failed to create profile: ' + profileErr.message }, { status: 500 })
}

const { error: contractorErr } = await admin.from('contractor_profiles').insert({ ... })
if (contractorErr) {
  await admin.auth.admin.deleteUser(authData.user.id)
  return NextResponse.json({ error: 'Failed to create contractor profile: ' + contractorErr.message }, { status: 500 })
}
```

---

### C-06 — Contractor Approve: DB Write Unchecked, Notification Goes Out Anyway
**File:** `src/app/api/contractors/[id]/approve/route.ts`  
**Lines:** ~21-27  
**Category:** Silent Supabase failure + SMS fire-and-forget  

**Current code:**
```ts
await admin.from('contractor_profiles').update({
  approval_status: 'approved',
  approved_by: user.id,
  approved_at: new Date().toISOString(),
}).eq('id', params.id)   // ← NO error check

// ... then SMS and email go out
if (up?.phone) await sendSMS(up.phone, `Your TAGS account is approved!...`)  // ← throws
if (up?.email) { ... await sendEmail(...) }
```

**Problem:** The approval update is not checked. If the DB write fails, the contractor receives an "approved" SMS/email but their `approval_status` remains `pending`. They log in, see they're still under review, and can't bid. `sendSMS` throws on failure — if phone is set and Twilio fails, the route returns 500 even though the approval (if it succeeded) already happened.

**Fix:**
```ts
const { error: approveErr } = await admin.from('contractor_profiles').update({
  approval_status: 'approved',
  approved_by: user.id,
  approved_at: new Date().toISOString(),
}).eq('id', params.id)

if (approveErr) return NextResponse.json({ error: 'Failed to approve contractor: ' + approveErr.message }, { status: 500 })

if (up?.phone) {
  try { await sendSMS(up.phone, `...`) } catch (e) { console.error('[approve] SMS failed:', e) }
}
```

---

### C-07 — Complete Project: Status Update Unchecked, SMS Throws
**File:** `src/app/api/projects/[id]/complete/route.ts`  
**Lines:** ~23-26, ~36-40  
**Category:** Silent Supabase failure + SMS fire-and-forget  

**Current code:**
```ts
await admin.from('projects').update({ status: 'completed', ... }).eq('id', params.id)
// ← NO error check — returns 200 even if update failed

if (contractor?.phone) {
  await sendSMS(contractor.phone, `Great news! ...`)  // ← throws on Twilio failure
}
```

**Problem:** If the `completed` update fails, the project stays `awarded` in the DB but the PM sees "marked complete" UI state (based on the 200 response). The contractor receives an SMS congratulating them on a job that isn't actually complete in the system. Rating flow is triggered prematurely. If SMS throws, completion still succeeded but user sees 500.

**Fix:** Check the update error, return 500 if it fails. Wrap SMS in try/catch.

---

### C-08 — Contractor Mark Done: Status Update Unchecked, SMS Loop Throws
**File:** `src/app/api/projects/[id]/contractor-complete/route.ts`  
**Lines:** ~28-30, ~39-43  
**Category:** Silent Supabase failure + SMS fire-and-forget  

**Current code:**
```ts
await admin.from('projects').update({ status: 'work_complete', ... }).eq('id', params.id)
// ← NO error check

for (const pm of (pmUsers || [])) {
  if (pm.phone) await sendSMS(pm.phone, `...`)  // ← throws inside loop — no try/catch
}
```

**Problem:** Same pattern as C-07. If there are 3 PMs and PM #1 SMS throws, PMs #2 and #3 are never notified. The route returns 500. The project status update may or may not have succeeded. Contractor sees an error and may re-submit.

**Fix:** Check update error. Wrap the SMS loop in individual try/catches.

---

## 🟠 HIGH

---

### H-01 — Bid Submission: PM Notification SMS Throws After Bid Is Written
**File:** `src/app/api/contractor/bids/route.ts`  
**Lines:** ~54-62  
**Category:** SMS/Email fire-and-forget  

**Current code:**
```ts
const { error: bidErr } = await admin.from('bids').insert({ ... })
if (bidErr) return NextResponse.json({ error: bidErr.message }, { status: 500 })

// Notify PM
for (const pm of (pmUsers || [])) {
  if (pm.phone) await sendSMS(pm.phone, msg)   // ← throws, crashes response
  if (pm.email) { ... await sendEmail(...) }    // ← sendEmail catches internally
}

return NextResponse.json({ success: true })
```

**Problem:** The bid IS written to the DB. But if PM SMS fails, `sendSMS` throws, and the contractor receives a 500 response. The contractor thinks their bid failed and may re-submit, creating a duplicate bid. The actual error ("bid already exists") will be confusing.

**Fix:**
```ts
for (const pm of (pmUsers || [])) {
  if (pm.phone) {
    try { await sendSMS(pm.phone, msg) } catch (e) { console.error('[bid] PM SMS failed:', e) }
  }
  if (pm.email) { ... }
}
```

---

### H-02 — SOW Generation: Save to DB Has No Error Check
**File:** `src/app/api/projects/generate-scope/route.ts`  
**Lines:** ~150-157  
**Category:** Silent Supabase failure  

**Current code:**
```ts
if (projectId && fullText) {
  const admin = createAdminClient()
  await admin.from('projects').update({
    scope_of_work: fullText,
    scope_generated_at: new Date().toISOString(),
  }).eq('id', projectId)
  // ← NO error check — fullText is streamed to client but may not be saved
}
```

**Problem:** The SOW streams successfully to the client (they see it rendered). But if the DB update fails, the SOW is not persisted. On page refresh, it's gone. The PM must regenerate. This is costly (Anthropic API call) and confusing. Because this runs inside a stream controller, errors are logged but not surfaced.

**Fix:**
```ts
const { error: saveErr } = await admin.from('projects').update({
  scope_of_work: fullText,
  scope_generated_at: new Date().toISOString(),
}).eq('id', projectId)

if (saveErr) {
  console.error('[SOW] Failed to save to DB:', saveErr)
  controller.enqueue(encoder.encode(`\n\n__SOW_SAVE_ERROR__: ${saveErr.message}`))
}
```

---

### H-03 — Stripe Checkout: No Try/Catch Around Stripe API Calls
**File:** `src/app/api/stripe/checkout/route.ts`  
**Lines:** ~75-120  
**Category:** Missing error handling  

**Current code:**
```ts
const customer = await stripe.customers.create({ ... })  // ← throws on Stripe error
// ...
const session = await stripe.checkout.sessions.create({ ... })  // ← throws on Stripe error
return NextResponse.json({ url: session.url })
```

**Problem:** Any Stripe API error (network issue, invalid price ID, API downtime) will bubble as an unhandled exception. Next.js will return a 500 with a generic error page, not a structured JSON response. The client has no actionable error to display.

Additionally:
```ts
await supabase.from('organizations').update({ stripe_customer_id: customerId }).eq('id', org.id)
// ← NO error check
```
If this update fails, the next checkout creates a duplicate Stripe customer, doubling charges risk.

**Fix:**
```ts
try {
  const customer = await stripe.customers.create({ ... })
  customerId = customer.id

  const { error: custErr } = await supabase.from('organizations')
    .update({ stripe_customer_id: customerId }).eq('id', org.id)
  if (custErr) console.error('Failed to save stripe_customer_id:', custErr)

  const session = await stripe.checkout.sessions.create({ ... })
  return NextResponse.json({ url: session.url })
} catch (err) {
  console.error('Stripe checkout error:', err)
  return NextResponse.json({ error: 'Checkout session creation failed. Please try again.' }, { status: 500 })
}
```

---

### H-04 — Stripe Portal: No Try/Catch Around Portal Session Creation
**File:** `src/app/api/stripe/portal/route.ts`  
**Lines:** ~35-40  
**Category:** Missing error handling  

**Current code:**
```ts
const portalSession = await stripe.billingPortal.sessions.create({
  customer: org.stripe_customer_id,
  return_url: `${baseUrl}/dashboard/billing`,
})
return NextResponse.json({ url: portalSession.url })
```

**Problem:** Same as H-03. If Stripe is unavailable or `stripe_customer_id` is stale/deleted, this throws an unhandled exception. The billing page shows a generic 500 instead of a user-facing message.

**Fix:** Wrap in try/catch, return structured JSON error.

---

### H-05 — Stripe Webhook: `invoice.payment_failed` Error Is Silently Dropped
**File:** `src/app/api/stripe/webhook/route.ts`  
**Lines:** ~78-88  
**Category:** Swallowed error + Silent Supabase failure  

**Current code:**
```ts
if (event.type === 'invoice.payment_failed') {
  const invoice = event.data.object as Stripe.Invoice
  const customerId = invoice.customer as string

  if (customerId) {
    const { error } = await supabase
      .from('organizations')
      .update({ subscription_status: 'past_due' })
      .eq('stripe_customer_id', customerId)

    if (error) {
      console.error('Webhook: failed to update past_due status', error)
      // ← NO error response returned — falls through to...
    }
  }
}

return NextResponse.json({ received: true })  // ← Always 200, even on DB failure
```

**Problem:** If the DB update fails, the webhook returns `200 received: true`. Stripe considers this event successfully delivered and will **not retry**. The organization's `subscription_status` stays `active` despite a failed payment. They retain full access when they shouldn't. This is a billing integrity hole.

**Fix:**
```ts
if (error) {
  console.error('Webhook: failed to update past_due status', error)
  return NextResponse.json({ error: 'DB update failed' }, { status: 500 })
  // Stripe will retry on 5xx
}
```

---

### H-06 — Contractor Reject: DB Write and SMS Both Unchecked
**File:** `src/app/api/contractors/[id]/reject/route.ts`  
**Lines:** ~20-29  
**Category:** Silent Supabase failure + SMS fire-and-forget  

**Current code:**
```ts
await admin.from('contractor_profiles').update({
  approval_status: 'rejected',
}).eq('id', params.id)  // ← NO error check

if (up?.phone) {
  await sendSMS(up.phone, 'Your TAGS application was not approved...')  // ← throws
}
```

**Problem:** If rejection fails silently, contractor still has `pending` status. If SMS throws, admin sees 500 even if rejection succeeded. Mirrors C-06.

---

### H-07 — Portfolio Connect: Invite Not Marked Used — Token Can Be Reused
**File:** `src/app/api/portfolio/connect/route.ts`  
**Lines:** ~38-45  
**Category:** Silent Supabase failure  

**Current code:**
```ts
const { error: joinErr } = await admin.from('portfolio_organizations').insert({ ... })
if (joinErr && !joinErr.message.includes('unique')) {
  return NextResponse.json({ error: joinErr.message }, { status: 500 })
}

// Mark invite as used
await admin.from('portfolio_invites')
  .update({ used_by_org_id: profile.organization_id })
  .eq('id', invite.id)   // ← NO error check
```

**Problem:** If the `portfolio_invites` update fails, the invite token remains valid. Any number of orgs can use the same token to join the portfolio. The PM organization joined, but the token is re-usable.

**Fix:**
```ts
const { error: markErr } = await admin.from('portfolio_invites')
  .update({ used_by_org_id: profile.organization_id })
  .eq('id', invite.id)
if (markErr) console.error('[portfolio/connect] Failed to mark invite used:', markErr)
// Non-fatal but should be logged — consider returning 500 if token security is important
```

---

### H-08 — New Project: No Validation That Property Belongs to PM's Org
**File:** `src/app/api/pm/projects/route.ts`  
**Lines:** ~22-34  
**Category:** Missing null check + authorization bypass  

**Current code:**
```ts
const { propertyId, ... } = await req.json()

const { data: property } = await admin.from('properties').select('*').eq('id', propertyId).single()
// ← No error check, no organization_id check

const { data: project, error } = await admin.from('projects').insert({
  organization_id: profile.organization_id,
  property_id: propertyId,   // ← Could be another org's property
  ...
}).select().single()
```

**Problem:** Any authenticated PM can create a project linked to any property in the database by passing a foreign `propertyId`. There's no check that the property belongs to their organization. The `property` fetch has no error check either — if `propertyId` is invalid, `property` is null but the insert proceeds anyway.

**Fix:**
```ts
const { data: property, error: propErr } = await admin.from('properties')
  .select('*')
  .eq('id', propertyId)
  .eq('organization_id', profile.organization_id)  // ← Enforce ownership
  .single()

if (propErr || !property) {
  return NextResponse.json({ error: 'Property not found or not owned by your organization' }, { status: 404 })
}
```

---

### H-09 — Stale Projects Cron: SMS Throws in Loop, Job Aborts
**File:** `src/app/api/admin/stale-projects/route.ts`  
**Lines:** ~37-43  
**Category:** SMS fire-and-forget  

**Current code:**
```ts
for (const pm of (pms || [])) {
  if (pm.phone) {
    await sendSMS(pm.phone, `Your project "${project.title}" has been live...`)  // ← throws
    alerted++
  }
}
```

**Problem:** If Twilio fails for any PM in any iteration, `sendSMS` throws, the loop exits, and remaining PMs and projects get no alert. The cron endpoint returns 500, giving no information about partial success.

**Fix:** Wrap in try/catch, increment `alerted` only on success, log failures.

---

### H-10 — SMS Inbound: Message Insert Has No Error Check — SMS Lost on DB Failure
**File:** `src/app/api/sms/inbound/route.ts`  
**Lines:** ~75-83  
**Category:** Silent Supabase failure  

**Current code:**
```ts
// Store the inbound message in the correct thread
await admin.from('project_messages').insert({
  project_id: projectId,
  sender_id: senderId,
  contractor_user_id: contractorUserId,
  message: body.trim(),
  channel: 'sms',
})   // ← NO error check — message silently lost if this fails
```

**Problem:** If the DB insert fails, the inbound SMS is silently dropped. Twilio receives a `200 OK` (no retry). The message is gone. The other party never sees it in the app thread. From the user's perspective, their SMS was sent and TAGS responded with silence.

**Fix:**
```ts
const { error: insertErr } = await admin.from('project_messages').insert({ ... })
if (insertErr) {
  console.error('[SMS inbound] Failed to store message:', insertErr)
  // Still return twiml() — Twilio can't help at this point, but log it
}
```

---

## 🟡 MEDIUM

---

### M-01 — Contractor Doc Upload: Signed URL Failure Stores Empty String
**File:** `src/app/api/upload/contractor-doc/route.ts`  
**Lines:** ~52-57  
**Category:** Missing null check + Silent Supabase failure  

**Current code:**
```ts
const { data: signedData } = await admin.storage
  .from('contractor-docs')
  .createSignedUrl(filename, 60 * 60 * 24 * 365 * 10)

const url = signedData?.signedUrl || ''  // ← Empty string if signedUrl failed

const { error: updateError } = await admin.from('contractor_profiles')
  .update({ [dbField]: url })             // ← Stores '' as compliance doc URL
  .eq('user_id', user.id)
```

**Problem:** `createSignedUrl` can fail (wrong bucket, permissions). The error is not checked. An empty string `''` is stored as the W9, COI, or license URL. The doc appears "uploaded" in the UI but the URL is broken. For compliance documents, this is a significant reliability issue.

**Fix:**
```ts
const { data: signedData, error: signedErr } = await admin.storage
  .from('contractor-docs')
  .createSignedUrl(filename, 60 * 60 * 24 * 365 * 10)

if (signedErr || !signedData?.signedUrl) {
  return NextResponse.json({ error: 'Failed to generate document URL' }, { status: 500 })
}
const url = signedData.signedUrl
```

---

### M-02 — Contractor Profile GET: Null Profile Returns Empty Object
**File:** `src/app/api/contractor/profile/route.ts`  
**Lines:** ~10-12, ~34  
**Category:** Missing null check  

**Current code:**
```ts
const { data } = await admin.from('contractor_profiles').select('*').eq('user_id', user.id).single()
// ← No error check

return NextResponse.json({ ...(data || {}), ratings, avgRating, ratingCount: ratings.length })
```

**Problem:** If the contractor has no profile (e.g., partial registration), `data` is null and `error` has details — but error is ignored. The response is `{ ratings: [], avgRating: null, ratingCount: 0 }` — an empty object. The profile page renders an empty/broken form with no error message.

**Fix:**
```ts
const { data, error } = await admin.from('contractor_profiles').select('*').eq('user_id', user.id).single()
if (error || !data) return NextResponse.json({ error: 'Contractor profile not found' }, { status: 404 })
```

---

### M-03 — PM Registration: SMS Throws After Account Is Created
**File:** `src/app/api/auth/register/pm/route.ts`  
**Lines:** ~44-46  
**Category:** SMS fire-and-forget  

**Current code:**
```ts
if (phone) await sendSMS(phone, `Welcome to TAGS!...`)   // ← throws on failure
return NextResponse.json({ success: true })
```

**Problem:** Account is created successfully but `sendSMS` throws. The registration route returns 500. The client shows a registration error. The user's account exists but they think registration failed. They'll try again and get "email already exists."

**Fix:**
```ts
if (phone) {
  try { await sendSMS(phone, `Welcome to TAGS!...`) } catch (e) { console.error('[pm/register] SMS failed:', e) }
}
```

---

### M-04 — Homeowner Registration: Address Property Insert Unchecked
**File:** `src/app/api/auth/register/homeowner/route.ts`  
**Lines:** ~36-44  
**Category:** Silent Supabase failure  

**Current code:**
```ts
if (address) {
  await admin.from('properties').insert({
    organization_id: org.id,
    created_by: authData.user.id,
    name: 'My Home',
    address,
    city: '',
    state: '',
    zip: '',
  })   // ← NO error check — address silently not saved
}
```

**Problem:** If property insert fails, registration returns success but the homeowner has no address property. Low severity for homeowner flow but inconsistent behavior.

---

### M-05 — ProjectMessages Component: Fetch Errors Are Silently Swallowed
**File:** `src/components/ProjectMessages.tsx`  
**Lines:** ~25-30  
**Category:** Client-side silent failure  

**Current code:**
```ts
async function fetchMessages() {
  const res = await fetch(`/api/projects/${projectId}/messages?contractorUserId=${contractorUserId}`)
  if (res.ok) {
    const data = await res.json()
    setMessages(data)
  }
  setLoading(false)   // ← No error state, no toast
}
```

**Problem:** If the API fails (auth expired, network error, server error), `setLoading(false)` is called and the UI shows an empty message box with no indication of failure. The polling continues every 10 seconds. No toast, no error state shown.

**Fix:**
```ts
async function fetchMessages() {
  try {
    const res = await fetch(...)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    const data = await res.json()
    setMessages(data)
  } catch (e) {
    console.error('[messages] fetch failed:', e)
    // On first load, show error; on polling, fail silently but stop updating
  } finally {
    setLoading(false)
  }
}
```

---

### M-06 — PreBidQA Component: Fetch Errors Silently Show Empty State
**File:** `src/components/PreBidQA.tsx`  
**Lines:** ~36-42  
**Category:** Client-side silent failure  

**Current code:**
```ts
async function fetchMessages() {
  const res = await fetch(`/api/projects/${projectId}/messages`)
  if (res.ok) {
    const data: Message[] = await res.json()
    // ... process threads
  }
  setLoading(false)   // ← No error handling
}
```

**Problem:** Identical to M-05. API failures render "No questions yet" even when there are active contractor threads. A PM may miss an urgent contractor question because the fetch silently failed.

---

### M-07 — SMS Inbound: Fetches ALL User Profiles for Phone Matching
**File:** `src/app/api/sms/inbound/route.ts`  
**Lines:** ~30-35  
**Category:** Missing null check + performance risk  

**Current code:**
```ts
const { data: allProfiles } = await admin
  .from('user_profiles')
  .select('id, full_name, role, phone')
// ← Fetches every user, no limit, no error check

const senderProfile = (allProfiles || []).find(p => ...)
```

**Problem:** As the user base grows, this fetches 1000s of rows on every inbound SMS to do an in-memory phone number match. At scale this will time out. No error check — if the query fails, `allProfiles` is null, `senderProfile` is undefined, and the SMS is dropped.

**Fix:** Index the `phone` column and query directly:
```ts
const normalized = normalizePhone(from)
const { data: senderProfile } = await admin
  .from('user_profiles')
  .select('id, full_name, role, phone')
  .eq('phone', normalized)
  .maybeSingle()
```
Note: phone normalization must be consistent at write time for this to work.

---

### M-08 — Publish: Accessing `project.properties.city` Without Null Check
**File:** `src/app/api/projects/[id]/publish/route.ts`  
**Lines:** ~52-54  
**Category:** Missing null check  

**Current code:**
```ts
const msg = `New ${project.project_type} project posted in ${project.properties.city}, ${project.properties.state}...`
```

**Problem:** `project.properties` is a join — if no property is linked to the project, `properties` is null. Accessing `.city` throws a TypeError and the entire publish request fails with an unhandled 500. The project remains unpublished.

**Fix:**
```ts
const city = project.properties?.city ?? 'your area'
const state = project.properties?.state ?? ''
const msg = `New ${project.project_type} project posted in ${city}${state ? ', ' + state : ''}...`
```

---

## 🔵 LOW

---

### L-01 — Invite Token Reuse: `contractor_invites.used_at` Update Not Checked
**File:** `src/app/api/auth/register/contractor/route.ts`  
**Lines:** ~69-72  
**Category:** Silent Supabase failure  

**Current code:**
```ts
if (inviteId) {
  await admin.from('contractor_invites')
    .update({ used_at: new Date().toISOString() })
    .eq('id', inviteId)   // ← NO error check
}
```

**Problem:** If this update fails, the invite token remains usable. A contractor can register multiple times with the same invite link, each time getting auto-approved status. Low risk since re-registration would fail on email uniqueness, but the token remains permanently valid.

---

### L-02 — Portfolio Invite Token Info: No Auth Check
**File:** `src/app/api/portfolio/invites/[token]/route.ts`  
**Lines:** ~1-35  
**Category:** Missing error handling / information disclosure  

**Current code:**
```ts
export async function GET(_req: NextRequest, { params }: ...) {
  // ← No auth check at all
  const { data: invite } = await admin.from('portfolio_invites').select(...)...

  return NextResponse.json({
    valid: true,
    portfolioName: portfolio.name,
    ownerName: owner?.full_name ?? 'Regional Manager',
  })
}
```

**Problem:** Anyone who guesses or obtains a token UUID can get the portfolio name and owner name. Since UUIDs are not guessable this is low risk, but the endpoint leaks PII (full name) without requiring authentication.

---

### L-03 — Update Existing Invite Phone: No Error Check
**File:** `src/app/api/invites/contractor/route.ts`  
**Lines:** ~45-49  
**Category:** Silent Supabase failure  

**Current code:**
```ts
if (phone) {
  await admin.from('contractor_invites')
    .update({ phone })
    .eq('inviting_org_id', profile.organization_id)
    .eq('email', email.toLowerCase())   // ← NO error check
}
```

**Problem:** If phone update fails, the invite is re-sent with the old phone number, meaning no SMS will be sent. No feedback to the PM.

---

### L-04 — Middleware: Auth Error Not Checked
**File:** `src/middleware.ts`  
**Lines:** ~22-24  
**Category:** Missing error handling  

**Current code:**
```ts
const { data: { user } } = await supabase.auth.getUser()
```

**Problem:** If `getUser()` throws (Supabase downtime, malformed JWT), the destructuring will throw an unhandled exception in middleware, which blocks ALL requests. Should be:
```ts
const { data: { user }, error: authErr } = await supabase.auth.getUser()
// If authErr, treat as unauthenticated and continue
```

---

### L-05 — Clipboard Write: No Error Handling in InviteContractorModal
**File:** `src/components/InviteContractorModal.tsx`  
**Lines:** ~48  
**Category:** Client-side silent failure  

**Current code:**
```ts
onClick={() => { navigator.clipboard.writeText(inviteUrl); toast.success('Link copied!') }}
```

**Problem:** `navigator.clipboard` is undefined in insecure contexts and on some mobile browsers. The copy fails silently but shows "Link copied!" toast.

**Fix:**
```ts
onClick={async () => {
  try {
    await navigator.clipboard.writeText(inviteUrl)
    toast.success('Link copied!')
  } catch {
    toast.error('Copy failed — please select and copy the link manually')
  }
}}
```

---

### L-06 — `lib/email.ts`: All Email Errors Are Silently Swallowed
**File:** `src/lib/email.ts`  
**Lines:** ~7-12  
**Category:** Swallowed errors  

**Current code:**
```ts
export async function sendEmail(to: string, subject: string, html: string) {
  if (!to || !process.env.RESEND_API_KEY) return
  try {
    const { Resend } = await import('resend')
    const resend = new Resend(process.env.RESEND_API_KEY)
    await resend.emails.send({ from: FROM, to, subject, html })
  } catch (e) {
    console.error('Email failed:', e)   // ← No return value, callers can't detect failure
  }
}
```

**Problem:** `sendEmail` always returns `undefined` (void). Callers cannot distinguish successful delivery from failure. Award notifications, invite emails, and approval emails fail silently with only a console.error. Consider returning a boolean or throwing so callers can handle failures appropriately.

---

### L-07 — PM Projects PATCH: No Field Whitelist on Update
**File:** `src/app/api/pm/projects/[id]/route.ts`  
**Lines:** PATCH handler  
**Category:** Missing validation  

**Current code:**
```ts
export async function PATCH(req: NextRequest, { params }: ...) {
  // ...
  const body = await req.json()
  const { error } = await admin.from('projects')
    .update({ ...body, updated_at: new Date().toISOString() })
    .eq('id', params.id)
    .eq('organization_id', profile.organization_id)
```

**Problem:** `body` is spread directly into the update without any field whitelist. A PM could pass `{ status: 'completed', created_by: 'other-user-id', organization_id: 'other-org' }` and overwrite sensitive fields. The `organization_id` check on the WHERE clause prevents cross-org access, but fields like `status`, `awarded_bid_id`, or `created_by` could be overwritten arbitrarily.

**Fix:** Explicitly pick only allowed fields:
```ts
const { title, description, budget_min, budget_max, unit_number } = await req.json()
const allowedUpdate = { title, description, budget_min, budget_max, unit_number, updated_at: new Date().toISOString() }
```

---

## Quick Reference: Files With No Issues Found

The following files were reviewed and had no significant silent failure patterns:

- `src/lib/supabase/admin.ts` — correctly throws on missing env var  
- `src/lib/supabase/server.ts` — standard SSR client setup  
- `src/lib/supabase/client.ts` — standard browser client setup  
- `src/lib/url.ts` — simple utility, no async  
- `src/lib/stripe.ts` — correctly throws on missing STRIPE_SECRET_KEY  
- `src/app/api/ratings/route.ts` — properly checks Supabase error  
- `src/app/api/user/profile/route.ts` — PATCH properly checks error  
- `src/app/api/pm/properties/route.ts` — properly checks insert error  
- `src/app/api/admin/view-as/route.ts` — properly checks all errors  
- `src/app/api/projects/[id]/data/route.ts` — properly checks error  
- `src/app/api/projects/[id]/messages/route.ts` — reasonable pattern  
- `src/components/AwardBidButton.tsx` — checks `res.ok`, shows toast  
- `src/components/AdminContractorActions.tsx` — checks `res.ok`, shows toast  
- `src/components/ConnectToPortfolio.tsx` — proper try/catch with error state  
- `src/components/InvitePropertyButton.tsx` — proper try/catch with error state  
- `src/components/ViewAsButton.tsx` — proper try/catch with toast  
- `src/components/CompleteProjectButton.tsx` — checks `res.ok`, shows toast  
- `src/components/ContractorMarkDoneButton.tsx` — checks `res.ok`, shows toast  
- `src/components/LeaveReviewButton.tsx` — checks `res.ok`, shows toast  
- `src/components/CompletionPhotoUpload.tsx` — checks `res.ok` per file  
- `src/middleware.ts` — minor concern logged as L-04  

---

## Priority Fix Order

Fix these before launch, in this sequence:

| Priority | Issue | Why |
|---|---|---|
| 1 | **C-01** | Photo uploads silently lost — permanent data loss |
| 2 | **C-02** | Award bid race condition — core transaction integrity |
| 3 | **C-03** | Award SMS crashes after writes — confuses PM, risks re-award |
| 4 | **C-04** | Publish fails silently — SMS sent to contractors for invisible project |
| 5 | **C-05** | Orphaned auth users on contractor registration |
| 6 | **C-06 / H-06** | Approve/Reject unchecked — notification without DB write |
| 7 | **H-05** | Stripe webhook payment_failed silently dropped |
| 8 | **H-03 / H-04** | Stripe checkout/portal — no try/catch on external API |
| 9 | **H-02** | Generated SOW not saved to DB — costly to reproduce |
| 10 | **H-01** | Bid submission — SMS throws after bid written, contractor retries |
| 11 | **C-07 / C-08** | Complete/contractor-complete status writes unchecked |
| 12 | **H-08** | PM can link project to another org's property |
| 13 | **H-10** | Inbound SMS lost on DB failure |
| 14 | **M-03** | PM registration SMS crashes after account created |
| 15 | **M-07** | Full user table scan on every inbound SMS |
