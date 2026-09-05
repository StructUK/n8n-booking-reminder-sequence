# SMS Reminders for Booking Confirmation & Reminder Sequence — Design

Date: 2026-09-05

## Summary

Extend the existing `n8n-booking-reminder-sequence` workflow to send SMS reminders (via Twilio) alongside the existing email reminders, at 24h and 2h before the appointment. This is v1 of the "SMS/WhatsApp booking reminders" idea noted 2026-09-05 — WhatsApp, two-way reply handling (cancel/reschedule), and standalone-repo packaging are explicitly out of scope for v1 (see Open Questions / Deferred below).

## Why

Health & wellness practices, trades, and appointment-based businesses lose money to no-shows. The existing workflow already confirms and reminds by email; SMS reaches customers who don't check email promptly and is the natural next channel, already flagged as a TODO in the existing README. Same trigger source and timing logic — no new intake path needed.

## Scope decisions (from brainstorming)

- **Extend the existing workflow**, not a new repo — reuses the booking intake, timing computation, and no-show tracking already built.
- **SMS only for v1** — no WhatsApp. No template-approval dependency, works for any phone number, fastest to stand up and demo.
- **Two-way reply handling is out of scope for v1** — send-only, matching the existing email reminder's scope. Documented as a "next step" in the README, same pattern used in `n8n-missed-call-whatsapp-followup`'s README.
- **Two SMS sends: 24h before and 2h before** the appointment.
- **Phone field becomes required** on the booking form (was optional) — a booking without a phone number can't receive the SMS reminders that are now a core part of this workflow.
- **No per-send Google Sheets logging for SMS**, matching the existing workflow's posture (email reminders aren't logged today either — only Confirmed / No-Show / Attended are logged).

## Architecture

The existing chain is:

```
New Booking Form → Parse Booking → Send Confirmation Email → Log (Confirmed)
  → Wait Until Reminder Time (24h before) → Send Reminder Email
  → Wait Until After Appointment (1h after) → Mark No-Show (Default) → Log (No-Show Check)
```

New chain:

```
New Booking Form → Parse Booking → Send Confirmation Email → Log (Confirmed)
  → Wait Until Reminder Time (24h before) → Send Reminder Email
                                           → Send Reminder SMS (24h)      [parallel branch]
  → Wait Until 2h Before                                                  [NEW]
  → Send Second Reminder SMS (2h)                                          [NEW]
  → Wait Until After Appointment (1h after) → Mark No-Show (Default) → Log (No-Show Check)
```

The "Mark Attended" webhook override path is unchanged.

## Components

### Form Trigger ("New Booking Form")
- Change `Phone (optional)` field to `Phone`, `requiredField: true`.

### Parse Booking (Code node, extended)
Add two computed fields to the existing return object:
- `reminder2TimeIso` — `apptTime - 2 hours`, ISO string (same pattern as existing `reminderTimeIso`/`checkTimeIso`).
- `phoneE164` — normalize the raw `Phone` field: strip spaces/dashes/parens; if it doesn't already start with `+`, assume a UK number and convert a leading `0` to `+44`. Documented as a UK-default assumption in the README (Struct Solutions is UK-based; non-UK users adjust this line).

### SMS Config (new Set node)
Holds `twilio_account_sid` and `twilio_from_number` as plain values, same pattern as the Config Values node in `n8n-missed-call-whatsapp-followup`. Feeds both new HTTP Request nodes. Runs once, in parallel with (or just before) the confirmation email — position it right after Parse Booking.

### Send Reminder SMS — 24h (new HTTP Request node)
- Twilio Messages API (`POST https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Messages.json`), Basic Auth credential (Account SID / Auth Token), same as the missed-call workflow's Twilio node — plain-text body needs no approved template, unlike WhatsApp.
- Body: `Hi {Customer Name}, reminder: your {Service} appointment is tomorrow at {time}. Reply if you need to reschedule.`
- Fed by "Wait Until Reminder Time", runs parallel to "Send Reminder Email".
- `onError: continueRegularOutput` — a Twilio failure doesn't break the Wait chain for that booking.

### Wait Until 2h Before (new Wait node)
- `resume: specificTime`, `dateTime: {{ $json.reminder2TimeIso }}`.
- Inserted between the 24h reminder step and the existing "Wait Until After Appointment".

### Send Second Reminder SMS — 2h (new HTTP Request node)
- Same Twilio call pattern as the 24h SMS.
- Body: `Hi {Customer Name}, your {Service} appointment is in 2 hours ({time}). See you soon!`
- Same `onError: continueRegularOutput` posture.

## Error handling

Both new Twilio HTTP Request nodes use `onError: continueRegularOutput`, matching the existing workflow's implicit tolerance for the email nodes — a failed SMS send never halts the Wait→Wait→no-show chain for that booking. No new failure-path logging (consistent with the current no-logging-for-reminders posture); if per-send visibility is wanted later, it's a documented extension, not part of v1.

## Testing

Manual acceptance test (added to README, same style as other workflows in this account):

1. Submit the booking form with an appointment time in the near future.
2. Confirm the confirmation email arrives immediately.
3. Confirm the 24h SMS arrives when "Wait Until Reminder Time" resumes (for testing, temporarily edit that Wait node's target time to a few minutes out — same caveat any Wait-based n8n workflow has for fast iteration).
4. Confirm the 2h SMS arrives when "Wait Until 2h Before" resumes.
5. Confirm the no-show check still fires afterward as before (regression check that the insert didn't break the existing tail).
6. Test with a malformed phone number — confirm the workflow logs the Twilio error via `continueRegularOutput` rather than crashing the execution, and email reminders still proceed normally.

No automated test suite — consistent with every other n8n workflow repo in this account; verification is manual, documented in the README.

## Setup additions (for README)

- Add a Twilio HTTP Basic Auth credential (Account SID / Auth Token), same as `n8n-missed-call-whatsapp-followup`.
- Fill in `twilio_account_sid` and `twilio_from_number` in the new SMS Config node.
- No WhatsApp template approval needed for this v1 (plain SMS).

## Open questions / deferred (not part of this design)

- **WhatsApp channel** — deferred; would reuse the Content Template pattern from `n8n-missed-call-whatsapp-followup` if added later.
- **Two-way reply handling** (cancel/reschedule via SMS reply) — deferred; would need an inbound SMS webhook and matching logic against the open booking, and is "a materially different, stateful workflow" per the missed-call README's own framing of the analogous WhatsApp-chatbot next step.
- **Showcase vs. client-offering positioning** — resolved implicitly: since this extends an existing showcase repo, it stays part of that repo rather than becoming a sixth separate showcase workflow.
- **Twilio SMS pricing** — to fold into fixed-price quoting; not a technical blocker for this build.
