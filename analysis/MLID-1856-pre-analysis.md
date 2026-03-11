# MLID-1856 — Pre-Analysis: Update timing and text of symptom text

**Analysis using Claude**

---

## Technical Breakdown

After a patient finishes an infusion appointment, the system sends them a follow-up SMS via Spruce asking about symptoms. The business wants **two changes**:

### Change 1: Update the SMS message text

**Where:** `apps/web/services/jobs/definitions/appointmentReminder/messageTemplate.ts` — function `symptomTextMessageFromTemplate()`

**Current text:**

> "Hello, it's been a few days since your treatment with Local Infusion. If you've experienced any symptoms..."

**New text (from ticket):**

> "Thank you for visiting Local Infusion! If you've experienced any symptoms, please do not hesitate to reach out for any questions and we will get back to you. For anything that is urgent, please reach out to your provider or 911. If not, please disregard. We have also provided some more information about discharge instructions in the link below. We hope you're doing well! :)"

Also update the **link URL** to the new HubSpot discharge instructions PDF provided in the ticket.

### Change 2: Change the send timing

**Where:** `apps/web/services/messages/messageService.ts` — function `setSymptomMessage()`

**Current behavior:**

- Scheduled 1 business day after the appointment
- Sent at **1:00 PM EST**

**New behavior:**

- Scheduled for the next business day after the appointment is completed
- Sent at **9:00 AM EST** instead of 1:00 PM

Key line to change: `dateToSend.setHours(13, 0, 0, 0)` → `dateToSend.setHours(9, 0, 0, 0)`

### How the system works (full picture)

1. **Trigger:** Looker webhook marks an appointment as "complete" → `webhookUpsertAppointments.ts` calls `setSymptomMessage()`
2. **Scheduling:** `setSymptomMessage()` creates a `Message` document in MongoDB with `source: SYMPTOM`, `status: READY`, and a `dateToSend` calculated as next business day
3. **Sending:** A background job (`sendMessageJob`) runs hourly 9AM-3PM EST, picks up READY messages whose `dateToSend` has passed, and sends them via Spruce SMS
4. **Guard:** Only fires for a patient's **first** completed appointment, gated by feature flag `SEND_SYMPTOM`

### Files to touch

| File | Change |
|------|--------|
| `messageTemplate.ts` | Update message text + URL |
| `messageService.ts` | Change hour from 13 to 9 in `setSymptomMessage()` |

### Scope

Small task — two string/number changes in two files. No new APIs, no UI changes, no new models.
