# SMS Reminders Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Twilio SMS reminders (24h and 2h before appointment) to the existing `n8n-booking-reminder-sequence` workflow, alongside the existing email reminders.

**Architecture:** Insert an "SMS Config" node and phone-normalization logic into the existing linear chain, fan out to a parallel Twilio HTTP Request SMS send at each of the two existing/new Wait resume points, deploy via the n8n API, then update the README. No new trigger, no new repo.

**Tech Stack:** n8n (self-hosted at `n8n.struct.solutions`), Twilio Messages API (SMS, plain text, HTTP Basic Auth), n8n MCP tools (`n8n_update_full_workflow`, `n8n_validate_workflow`) for deployment.

## Global Constraints

- Workflow ID in the live n8n instance: `vNrQQGzt1jjPpai6` (name "Booking Confirmation & Reminder Sequence").
- Phone field on the form becomes **required** (was optional).
- Phone normalization assumes **UK numbers** by default (leading `0` → `+44`) if the entered number doesn't already start with `+`.
- SMS sends use `onError: continueRegularOutput` — a Twilio failure must never break the Wait chain.
- No new Google Sheets logging for SMS sends (matches existing no-logging-for-reminders posture).
- Credential name for Twilio must be exactly `Twilio Basic Auth account` (matches the credential already documented/used in the sibling `n8n-missed-call-whatsapp-followup` repo, so a user who's already set up that credential can reuse it here).

---

### Task 1: Update `workflow.json` with SMS nodes and phone handling

**Files:**
- Modify: `E:\claude\projects\n8n projects\n8n-booking-reminder-sequence\workflow.json` (full rewrite — most of the file changes)

**Interfaces:**
- Produces: node names `SMS Config`, `Send Reminder SMS (24h)`, `Wait Until 2h Before`, `Send Second Reminder SMS (2h)` — referenced in Task 2's deploy call and Task 3's README.
- Produces: `Parse Booking` code node now emits `reminder2TimeIso` and `phoneE164` fields, consumed by the new Wait/SMS nodes.

- [ ] **Step 1: Replace the full contents of `workflow.json`**

```json
{
  "name": "Booking Confirmation & Reminder Sequence",
  "nodes": [
    {
      "id": "booking_form",
      "name": "New Booking Form",
      "type": "n8n-nodes-base.formTrigger",
      "typeVersion": 2.3,
      "position": [0, 0],
      "parameters": {
        "formTitle": "Book an appointment",
        "formDescription": "Confirm your details and we'll send you a confirmation and reminder.",
        "path": "new-booking",
        "formFields": {
          "values": [
            { "fieldLabel": "Customer Name", "fieldType": "text", "requiredField": true },
            { "fieldLabel": "Email", "fieldType": "email", "requiredField": true },
            { "fieldLabel": "Phone", "fieldType": "text", "requiredField": true },
            { "fieldLabel": "Service", "fieldType": "text", "requiredField": true },
            { "fieldLabel": "Appointment Date & Time (YYYY-MM-DD HH:mm)", "fieldType": "text", "requiredField": true }
          ]
        }
      }
    },
    {
      "id": "parse_booking",
      "name": "Parse Booking",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [220, 0],
      "parameters": {
        "jsCode": "const d = $input.first().json;\nconst apptTime = new Date(String(d['Appointment Date & Time (YYYY-MM-DD HH:mm)']).replace(' ', 'T'));\nconst bookingId = `${(d['Email']||'').toLowerCase()}-${apptTime.getTime()}`;\nconst reminderTime = new Date(apptTime.getTime() - 24*60*60*1000);\nconst reminder2Time = new Date(apptTime.getTime() - 2*60*60*1000);\nconst checkTime = new Date(apptTime.getTime() + 60*60*1000);\nlet phoneE164 = String(d['Phone'] || '').replace(/[\\s\\-()]/g, '');\nif (phoneE164 && !phoneE164.startsWith('+')) {\n  phoneE164 = phoneE164.replace(/^0/, '+44');\n}\nreturn [{ json: { ...d, bookingId, apptTimeIso: apptTime.toISOString(), reminderTimeIso: reminderTime.toISOString(), reminder2TimeIso: reminder2Time.toISOString(), checkTimeIso: checkTime.toISOString(), phoneE164, status: 'Confirmed' } }];"
      }
    },
    {
      "id": "sms_config",
      "name": "SMS Config",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [440, 0],
      "parameters": {
        "mode": "manual",
        "assignments": {
          "assignments": [
            { "id": "sc1", "name": "twilio_account_sid", "value": "REPLACE_WITH_TWILIO_ACCOUNT_SID", "type": "string" },
            { "id": "sc2", "name": "twilio_from_number", "value": "REPLACE_WITH_TWILIO_FROM_NUMBER", "type": "string" }
          ]
        },
        "includeOtherFields": true,
        "options": {}
      }
    },
    {
      "id": "send_confirmation",
      "name": "Send Confirmation Email",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [660, 0],
      "parameters": {
        "fromEmail": "hello@struct.solutions",
        "toEmail": "={{$json['Email']}}",
        "subject": "={{ 'Booking confirmed: ' + $json['Service'] }}",
        "emailFormat": "text",
        "text": "={{ 'Hi ' + $json['Customer Name'] + ',\\n\\nYour booking for ' + $json['Service'] + ' on ' + $json['Appointment Date & Time (YYYY-MM-DD HH:mm)'] + ' is confirmed. We will send you a reminder 24 hours beforehand.\\n\\nSee you then!' }}",
        "options": {}
      },
      "credentials": { "smtp": { "name": "SMTP account" } }
    },
    {
      "id": "log_confirmed",
      "name": "Log Booking (Confirmed)",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.5,
      "position": [880, 0],
      "parameters": {
        "operation": "append",
        "documentId": { "__rl": true, "mode": "id", "value": "REPLACE_WITH_SPREADSHEET_ID" },
        "sheetName": { "__rl": true, "mode": "name", "value": "Bookings" },
        "columns": { "mappingMode": "autoMapInputData", "matchingColumns": [], "schema": [], "value": {} },
        "options": {}
      },
      "credentials": { "googleSheetsOAuth2Api": { "name": "Google Sheets account" } }
    },
    {
      "id": "wait_reminder",
      "name": "Wait Until Reminder Time",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1.1,
      "position": [1100, 0],
      "parameters": { "resume": "specificTime", "dateTime": "={{ $json.reminderTimeIso }}" }
    },
    {
      "id": "send_reminder",
      "name": "Send Reminder Email",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [1320, -100],
      "parameters": {
        "fromEmail": "hello@struct.solutions",
        "toEmail": "={{$json['Email']}}",
        "subject": "={{ 'Reminder: ' + $json['Service'] + ' tomorrow' }}",
        "emailFormat": "text",
        "text": "={{ 'Hi ' + $json['Customer Name'] + ',\\n\\nJust a reminder that your appointment for ' + $json['Service'] + ' is coming up on ' + $json['Appointment Date & Time (YYYY-MM-DD HH:mm)'] + '.\\n\\nSee you soon!' }}",
        "options": {}
      },
      "credentials": { "smtp": { "name": "SMTP account" } }
    },
    {
      "id": "send_reminder_sms_24h",
      "name": "Send Reminder SMS (24h)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.3,
      "position": [1320, 100],
      "onError": "continueRegularOutput",
      "parameters": {
        "method": "POST",
        "url": "=https://api.twilio.com/2010-04-01/Accounts/{{ $json.twilio_account_sid }}/Messages.json",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpBasicAuth",
        "sendBody": true,
        "contentType": "form-urlencoded",
        "bodyParameters": {
          "parameters": [
            { "name": "To", "value": "={{ $json.phoneE164 }}" },
            { "name": "From", "value": "={{ $json.twilio_from_number }}" },
            { "name": "Body", "value": "={{ 'Hi ' + $json['Customer Name'] + ', reminder: your ' + $json['Service'] + ' appointment is tomorrow at ' + $json['Appointment Date & Time (YYYY-MM-DD HH:mm)'] + '. Reply if you need to reschedule.' }}" }
          ]
        },
        "options": {}
      },
      "credentials": { "httpBasicAuth": { "name": "Twilio Basic Auth account" } }
    },
    {
      "id": "wait_2h_before",
      "name": "Wait Until 2h Before",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1.1,
      "position": [1540, -100],
      "parameters": { "resume": "specificTime", "dateTime": "={{ $json.reminder2TimeIso }}" }
    },
    {
      "id": "send_reminder_sms_2h",
      "name": "Send Second Reminder SMS (2h)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.3,
      "position": [1760, 100],
      "onError": "continueRegularOutput",
      "parameters": {
        "method": "POST",
        "url": "=https://api.twilio.com/2010-04-01/Accounts/{{ $json.twilio_account_sid }}/Messages.json",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpBasicAuth",
        "sendBody": true,
        "contentType": "form-urlencoded",
        "bodyParameters": {
          "parameters": [
            { "name": "To", "value": "={{ $json.phoneE164 }}" },
            { "name": "From", "value": "={{ $json.twilio_from_number }}" },
            { "name": "Body", "value": "={{ 'Hi ' + $json['Customer Name'] + ', your ' + $json['Service'] + ' appointment is in 2 hours (' + $json['Appointment Date & Time (YYYY-MM-DD HH:mm)'] + '). See you soon!' }}" }
          ]
        },
        "options": {}
      },
      "credentials": { "httpBasicAuth": { "name": "Twilio Basic Auth account" } }
    },
    {
      "id": "wait_checkin",
      "name": "Wait Until After Appointment",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1.1,
      "position": [1760, -100],
      "parameters": { "resume": "specificTime", "dateTime": "={{ $json.checkTimeIso }}" }
    },
    {
      "id": "mark_noshow_default",
      "name": "Mark No-Show (Default)",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [1980, -100],
      "parameters": {
        "includeOtherFields": true,
        "assignments": {
          "assignments": [
            { "id": "1", "name": "status", "type": "string", "value": "No-Show (auto - overwritten if Mark Attended was called)" }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "log_noshow",
      "name": "Log Booking (No-Show Check)",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.5,
      "position": [2200, -100],
      "parameters": {
        "operation": "append",
        "documentId": { "__rl": true, "mode": "id", "value": "REPLACE_WITH_SPREADSHEET_ID" },
        "sheetName": { "__rl": true, "mode": "name", "value": "Bookings" },
        "columns": { "mappingMode": "autoMapInputData", "matchingColumns": [], "schema": [], "value": {} },
        "options": {}
      },
      "credentials": { "googleSheetsOAuth2Api": { "name": "Google Sheets account" } }
    },
    {
      "id": "mark_attended_webhook",
      "name": "Mark Attended (Webhook)",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2.1,
      "position": [220, 220],
      "parameters": { "path": "mark-attended", "httpMethod": "POST", "responseMode": "onReceived", "options": {} }
    },
    {
      "id": "log_attended",
      "name": "Log Booking (Attended)",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.5,
      "position": [440, 220],
      "parameters": {
        "operation": "append",
        "documentId": { "__rl": true, "mode": "id", "value": "REPLACE_WITH_SPREADSHEET_ID" },
        "sheetName": { "__rl": true, "mode": "name", "value": "Bookings" },
        "columns": { "mappingMode": "autoMapInputData", "matchingColumns": [], "schema": [], "value": {} },
        "options": {}
      },
      "credentials": { "googleSheetsOAuth2Api": { "name": "Google Sheets account" } }
    }
  ],
  "connections": {
    "New Booking Form": { "main": [[{ "index": 0, "node": "Parse Booking", "type": "main" }]] },
    "Parse Booking": { "main": [[{ "index": 0, "node": "SMS Config", "type": "main" }]] },
    "SMS Config": { "main": [[{ "index": 0, "node": "Send Confirmation Email", "type": "main" }]] },
    "Send Confirmation Email": { "main": [[{ "index": 0, "node": "Log Booking (Confirmed)", "type": "main" }]] },
    "Log Booking (Confirmed)": { "main": [[{ "index": 0, "node": "Wait Until Reminder Time", "type": "main" }]] },
    "Wait Until Reminder Time": { "main": [[
      { "index": 0, "node": "Send Reminder Email", "type": "main" },
      { "index": 0, "node": "Send Reminder SMS (24h)", "type": "main" }
    ]] },
    "Send Reminder Email": { "main": [[{ "index": 0, "node": "Wait Until 2h Before", "type": "main" }]] },
    "Wait Until 2h Before": { "main": [[
      { "index": 0, "node": "Send Second Reminder SMS (2h)", "type": "main" },
      { "index": 0, "node": "Wait Until After Appointment", "type": "main" }
    ]] },
    "Wait Until After Appointment": { "main": [[{ "index": 0, "node": "Mark No-Show (Default)", "type": "main" }]] },
    "Mark No-Show (Default)": { "main": [[{ "index": 0, "node": "Log Booking (No-Show Check)", "type": "main" }]] },
    "Mark Attended (Webhook)": { "main": [[{ "index": 0, "node": "Log Booking (Attended)", "type": "main" }]] }
  },
  "pinData": {},
  "settings": {
    "executionOrder": "v1"
  }
}
```

- [ ] **Step 2: Verify the file is valid JSON**

Run: `node -e "JSON.parse(require('fs').readFileSync('workflow.json','utf8')); console.log('OK')"`
Expected: `OK`
(If `node` isn't available, use: `python -c "import json; json.load(open('workflow.json')); print('OK')"`)

- [ ] **Step 3: Commit**

```bash
git add workflow.json
git commit -m "Add SMS reminders (24h + 2h before) via Twilio"
```

---

### Task 2: Deploy to the live n8n instance and validate

**Files:** none (uses the `workflow.json` content from Task 1 via n8n MCP tools — no local file changes)

**Interfaces:**
- Consumes: the exact `nodes` array and `connections` object written in Task 1, Step 1.
- Consumes: workflow ID `vNrQQGzt1jjPpai6` (from Global Constraints).

- [ ] **Step 1: Push the updated workflow**

Call `mcp__MCP_DOCKER__n8n_update_full_workflow` with:
- `id`: `vNrQQGzt1jjPpai6`
- `name`: `Booking Confirmation & Reminder Sequence`
- `nodes`: the full `nodes` array from Task 1 Step 1
- `connections`: the full `connections` object from Task 1 Step 1

Expected: response has `success: true`.

- [ ] **Step 2: Validate the deployed workflow**

Call `mcp__MCP_DOCKER__n8n_validate_workflow` with `id: "vNrQQGzt1jjPpai6"`, `options: { profile: "runtime" }`.

Expected: no `errors` in the response (warnings about missing credentials/placeholder IDs like `REPLACE_WITH_TWILIO_ACCOUNT_SID` are expected and fine — those are filled in by the end user per the README, same as every other workflow in this account).

- [ ] **Step 3: If validation reports structural errors (bad connections, unknown node type, expression syntax)**

Fix `workflow.json` (Task 1 file), re-run Task 1 Step 2 (JSON check), then repeat Task 2 Step 1–2 until clean. Do not proceed to Task 3 until validation is clean of structural errors.

---

### Task 3: Update README and push

**Files:**
- Modify: `E:\claude\projects\n8n projects\n8n-booking-reminder-sequence\README.md`

**Interfaces:** none (documentation only)

- [ ] **Step 1: Replace the full contents of `README.md`**

```markdown
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
```

- [ ] **Step 2: Commit and push**

```bash
git add README.md
git commit -m "Document SMS reminder setup and acceptance test"
git push
```
