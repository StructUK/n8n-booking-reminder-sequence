# n8n Booking Confirmation & Reminder Sequence

Confirms a new booking the moment it's made, sends email *and SMS* reminders ahead of the appointment, and flags no-shows automatically — no manual chasing.

Built by [Struct Solutions](https://struct.solutions), a UK-based AI automation consultancy — fixed-price n8n + AI workflow builds from £250.

![Workflow diagram](workflow-diagram.svg)

## Who this is for

**Health & wellness practices** (clinics, salons, therapists) and **trades businesses** that take scheduled appointments and lose time to no-shows or forgotten bookings. Also a good fit for any professional-services business that books calls or site visits.

## How it works

### Main sequence (triggered by a new booking)
- **Form Trigger** ("New Booking Form") — captures Customer Name, Email, Phone, Service and the appointment date/time. Phone is required, since it's now used for SMS reminders. Swap this for a webhook if you already have a booking widget (Calendly, Acuity, a custom site form) that can POST to n8n instead.
- **Code** ("Parse Booking") — parses the appointment time, computes three future timestamps (24h and 2h before, for the two reminders; 1h after, for the no-show check), and normalizes the phone number to E.164 (assumes a UK number if it doesn't already start with `+`, converting a leading `0` to `+44`).
- **Set** ("SMS Config") — your Twilio Account SID and SMS-sending number, in one place.
- **Send Email** — sends an immediate confirmation.
- **Google Sheets** — logs the booking as "Confirmed".
- **Wait** (until 24h before the appointment) — n8n's Wait node pauses this specific execution without tying up a worker, then resumes automatically at the exact time.
- **Send Email** + **HTTP Request** ("Send Reminder SMS (24h)") — sends the 24h reminder on both channels. The SMS uses Twilio's Messages API directly with a plain-text body (no template approval needed, unlike WhatsApp).
- **Wait** (until 2h before the appointment) — pauses again.
- **HTTP Request** ("Send Second Reminder SMS (2h)") — sends a second, closer-to-the-time SMS reminder.
- **Wait** (until 1h after the appointment) — pauses again.
- **Set + Google Sheets** — logs the booking as "No-Show (auto)" by default.

Both SMS sends use `onError: continueRegularOutput` — a Twilio failure (bad number, account issue, etc.) never crashes the execution or blocks the rest of the sequence; it's simply not delivered. No separate SMS delivery log is kept, matching this workflow's existing email reminders (only Confirmed / No-Show / Attended are logged to the sheet).

### Override path (triggered separately, any time)
- **Webhook** (`POST /mark-attended`) — call this from your booking system, calendar, or a simple staff-facing button any time after the appointment happens, with the booking's email/time in the body.
- **Google Sheets** — logs an "Attended" row.

Because the sheet is treated as an append-only event log, you get a clear timeline per booking (Confirmed → Reminder implied by timing → Attended *or* No-Show). Filter/pivot in the spreadsheet to see current status per booking.

**Why an event log instead of updating one row:** it's simpler to build and impossible to lose an update to a race condition between the "mark attended" call and the automatic no-show check. The tradeoff is you do a small pivot/filter in the sheet to see current state — documented here so it's a deliberate choice, not a gap.

## What it connects to

| Service | Used for | Required? |
|---|---|---|
| SMTP (email) | confirmation + reminder emails | Yes |
| Twilio (SMS) | 24h and 2h reminder texts | Yes |
| Google Sheets | booking/status log | Yes |

No AI/LLM calls in this workflow — it's a pure scheduling/notification sequence.

## Setup

1. Import `workflow.json` into your n8n instance.
2. Add your SMTP credential to both **Send Confirmation Email** and **Send Reminder Email**.
3. Add a Twilio HTTP Basic Auth credential named **"Twilio Basic Auth account"** (username = Account SID, password = Auth Token) to both **Send Reminder SMS (24h)** and **Send Second Reminder SMS (2h)**.
4. In the **SMS Config** node, replace `REPLACE_WITH_TWILIO_ACCOUNT_SID` with your Twilio Account SID and `REPLACE_WITH_TWILIO_FROM_NUMBER` with your Twilio SMS-capable phone number (E.164 format, e.g. `+15558675309`).
5. Add your Google Sheets credential to all three Google Sheets nodes, and point `documentId` at a spreadsheet with a "Bookings" sheet.
6. Replace `hello@struct.solutions` with your own sending address.
7. If most of your customers aren't in the UK, adjust the phone-normalization line in **Parse Booking** (`phoneE164 = phoneE164.replace(/^0/, '+44')`) for your country's dialing code, or require customers to enter numbers already in `+`-prefixed E.164 format.
8. Activate the workflow. Share the Form Trigger URL as your booking form, or point an existing booking system's webhook at the same intake logic.
9. Wire up "mark attended" however suits you: a button in your admin panel, a Zapier/Make step from your calendar, or just a quick manual POST request after each appointment.

### Manual acceptance test (do this after step 8)

1. Submit the booking form with an appointment time a few minutes in the future, and a real phone number you can check.
2. Confirm the confirmation email arrives immediately.
3. Temporarily edit the **Wait Until Reminder Time** node's target time to a couple of minutes out, save, and re-trigger (or wait for a real 24h-out booking). Confirm both the reminder email and the 24h SMS arrive.
4. Temporarily edit the **Wait Until 2h Before** node's target time the same way. Confirm the second SMS arrives.
5. Let (or force) **Wait Until After Appointment** resume, and confirm a "No-Show (auto...)" row still appears in the sheet as before — this checks the insert didn't break the existing tail of the workflow.
6. Test with a deliberately malformed phone number (e.g. letters instead of digits) — confirm the workflow doesn't crash: the SMS send fails silently (`onError: continueRegularOutput`) and the email reminders still go out normally.

### TODO / left for you

- Connect your own SMTP, Twilio, and Google Sheets credentials (not exported, for security) and the real spreadsheet ID.
- If you have an existing booking system with its own webhook, replace the Form Trigger with a Webhook node and map its fields instead.
- Consider adding WhatsApp as an additional channel — the `n8n-missed-call-whatsapp-followup` workflow in this account has the Twilio WhatsApp Content Template pattern to reuse, if you want WhatsApp support later (needs an approved message template, unlike this SMS-only build).
- Two-way reply handling (customer texts "C" to cancel / "R" to reschedule) isn't built here — it would need an inbound SMS webhook and matching logic against the open booking, and is a materially different, stateful piece of work worth its own design pass.

## License

MIT — see [LICENSE](LICENSE).
