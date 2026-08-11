# SMS Feature Complete

## Status: ✅ Shipped (committed to main, Netlify auto-deploying)

## What was built

### Database
- `project_messages.channel` — `text NOT NULL DEFAULT 'app'` (values: `'app'` | `'sms'`)
- `user_profiles.phone` — `text` (nullable, stores E.164 phone numbers)
- Both migrations run via Supabase Management API on project `sdjhnjuyfgwnjubzsfid`

### Inbound SMS webhook — `/api/sms/inbound`
- Twilio POSTs here (form-encoded) when someone texts +19139149928
- Parses `From` + `Body` via `req.formData()` with fallback URLSearchParams
- Normalizes phone to E.164, looks up `user_profiles.phone`
- Not found → TwiML error message
- Found → finds active project (`open` or `awarded`)
  - Multiple projects → TwiML lists them, asks user to use app
  - Single project → inserts `project_messages` with `channel='sms'`, then SMSes the other party
- PM → notifies awarded contractor; Contractor → notifies PM

### Outbound bridge — `/api/projects/[id]/messages` POST
- Saves messages with `channel='app'`
- After saving: looks up other party's phone
  - PM sends → notifies awarded contractor by SMS
  - Contractor sends → notifies PM by SMS
- SMS format: `"[Name] via TAGS ([Project Title]): [message]. Reply to this text to respond."`
- SMS failure is non-blocking (caught silently, doesn't break API response)

### ProjectMessages component
- `Message` interface now includes `channel: 'app' | 'sms'`
- SMS messages show a grey pill badge ("SMS") next to the timestamp

## Key schema detail
- Bids table uses `contractor_user_id` (not `contractor_id`)

## Git
- Commit: `feat: two-way SMS messaging — inbound webhook, outbound bridge, unified thread`
- Branch: `main` — up to date with origin

## Twilio config (env vars required)
- `TWILIO_ACCOUNT_SID`
- `TWILIO_AUTH_TOKEN`
- `TWILIO_FROM_NUMBER=+19139149928`
- `TWILIO_MESSAGING_SERVICE_SID` (preferred for A2P compliance)

## Next steps (not built)
- Multi-project SMS selection flow (requires server-side session state per phone number)
- Phone number capture UI in PM/contractor profile settings
- Twilio webhook URL configuration: point to `https://<domain>/api/sms/inbound`
