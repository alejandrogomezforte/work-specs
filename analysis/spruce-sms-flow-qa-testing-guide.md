# Spruce SMS Flow — QA Testing Guide

## Overview

This guide explains how to manually test the full Spruce SMS flow end-to-end in staging. The flow covers: appointment creation → message creation → SMS delivery via Spruce → conversation archival.

**Staging base URL**: `https://app-stage.mylocalinfusion.ai`

---

## Step 1: Create the appointment via webhook

**Endpoint**: `POST /api/webhooks/looker_appointments`

**Auth header**: `x-looker-webhook-token: <token>` (ask the team for the staging token)

This webhook simulates how Looker sends appointment data to the app. To trigger the SMS flow, we need to call it **twice** — first to create the appointment, then to transition it to `Complete`.

### Important notes about the payload

- **`appointments.id`**: Must be a unique numeric ID. Use the `99999xxx` range for testing (e.g., `99999931`, `99999932`, etc.). **Increment this value every time you run a new test** — reusing an ID will update the existing appointment instead of creating a new one.
- **Dates**: Adjust all dates to match today's date so the message is created for the current time window:
  - `appointments.start_local_tz_time`: Today's date with a morning time (e.g., `"2026-04-21 10:30:00"`)
  - `appointments.created_date`: Today's date (e.g., `"2026-04-21"`)
  - `orders.created_date_reformat`: Any recent past date (e.g., a week ago)
- **`orders.patient_id`**: Must match an existing patient's `we_infuse_id_IV` in the `patients` collection. Use `1013636` for testing.
- **`locations.id`**: Must match an existing location. Use `1834` for testing.
- **`patients.mobile_phone`**: The phone number that will receive the SMS. Use `9176755175` for testing.

### Send the webhook with status `Complete`

A single POST with status `Complete` is enough. When the appointment doesn't exist yet, the insert path in the webhook handler detects the `Complete` status and creates the message directly — no need to first create it as `Active`.

```bash
curl -X POST https://app-stage.mylocalinfusion.ai/api/webhooks/looker_appointments \
  -H "Content-Type: application/json" \
  -H "x-looker-webhook-token: YOUR_TOKEN_HERE" \
  -d '{
    "data": [
      {
        "appointments.id": "99999932",
        "locations.id": "1834",
        "appointments.status": "Complete",
        "orders.patient_id": "1013636",
        "group_providers.npi": "1234567890",
        "appointments.start_local_tz_time": "YYYY-MM-DD 10:30:00",
        "orders.created_date_reformat": "YYYY-MM-DD",
        "patients.mobile_phone": "9176755175",
        "patients.home_phone": "9176755175",
        "appointments.created_date": "YYYY-MM-DD",
        "appointments.order_series_id": 1001,
        "appointments.first_appt_on_order_series": "No",
        "orders.first_listed_primary_med": ""
      }
    ]
  }'
```

> Replace `YYYY-MM-DD` with today's date (e.g., `2026-04-21`).
> Replace `99999932` with an unused ID.

### Optional: Two-call approach (Active → Complete)

If you want to simulate the real-world flow where Looker sends status updates over time, you can optionally send two calls: first with `"appointments.status": "Active"`, then the same payload with `"appointments.status": "Complete"`. The app also creates the message on the `Active → Complete` transition (update path). Both approaches produce the same result.

### How to verify

After the webhook call, check the `messages` collection in MongoDB. You should see a new document with:

- `source: "symptom"`
- `status: "ready"`
- `appointmentId` matching the newly created appointment

---

## Step 2: Set the message for immediate delivery

By default, the message's `dateToSend` is set to the **next business day at 1 PM UTC** (9 AM ET), and the job only sends messages during work hours (9 AM–3 PM ET). To bypass these constraints and send immediately, update the message in MongoDB.

### Find the message

In MongoDB, look for the message created in Step 1. You can filter by the appointment's `we_infuse_id`:

```javascript
// First, find the appointment to get its _id
db.appointments.findOne({ we_infuse_id: 99999932 })

// Then find the message using the appointment's _id
db.messages.findOne({ appointmentId: ObjectId("<appointment_id>"), source: "symptom" })
```

### Update the message for immediate delivery

```javascript
db.messages.updateOne(
  { _id: ObjectId("<message_id>") },
  { $set: { immediately: true, dateToSend: new Date() } }
)
```

Replace `<message_id>` with the `_id` of the message document found above.

This does two things:

- `immediately: true` — bypasses the work hours check and holiday rescheduling
- `dateToSend: new Date()` — sets the send date to right now so the job picks it up

---

## Step 3: Trigger the send message job

Go to the **staging web app** → **Admin** → **Jobs** (`/admin/jobs`).

Find the job named **`sendMessageProcessing`** and trigger it.

The job will:

1. Pick up the message (because `immediately: true` bypasses all time guards)
2. Send the SMS via the Spruce API
3. Spruce will fire a webhook back to the app (`/api/webhooks/spruce_conversations`)
4. The webhook handler will link the conversation to the message and archive it

---

## Step 4: Verify the full flow

After triggering the job, check the following in MongoDB:

### Message document (`messages` collection)

| Field | Expected value |
|-------|---------------|
| `status` | `sent` |
| `spruce.requestId` | Populated (Spruce API response ID) |
| `spruce.conversationId` | Populated (from Spruce webhook callback) |
| `spruce.appUrl` | Populated (Spruce conversation URL) |

### Conversation document (`conversations` collection)

A new document should exist with the Spruce conversation data.

### ConversationItem document (`conversationitems` collection)

A new document should exist with the outbound message details.

### Appointment document (`appointments` collection)

The appointment's `reminder` field may be updated with the send status.

---

## Quick Reference: Test Data

| Field | Value | Notes |
|-------|-------|-------|
| Patient ID | `1013636` | Must exist in `patients` collection |
| Location ID | `1834` | Must exist in `patientLocations` collection |
| Phone | `9176755175` | Test phone number |
| Appointment ID range | `99999xxx` | Increment for each test |
| NPI | `1234567890` | Test provider NPI |

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Webhook returns 403 | Invalid or missing `x-looker-webhook-token` | Check the token value |
| No message created after Step 1b | Appointment didn't transition from Active → Complete | Verify Step 1a created the appointment first |
| Job runs but message stays `ready` | `SEND_SYMPTOM` feature flag is disabled | Enable it in Admin → Feature Flags |
| Job runs but message gets `declined` | Patient location not found | Verify patient and location exist in the DB |
| SMS sent but not archived | Archive feature flag disabled | Enable `SPRUCE_SMS_REMINDER_ARCHIVE` in Feature Flags |
