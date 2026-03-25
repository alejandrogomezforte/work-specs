# [MLID-1856] Update Checkout SMS Timing, Text, and Archival

## Summary

- Update the "symptom" SMS to function as a "checkout" message with new discharge instructions text and HubSpot URL
- Send a checkout SMS after **every** completed appointment, not just the first one
- Fix `dateToSend` calculation to anchor to the **appointment date** instead of the webhook execution time
- Generalize Spruce conversation archival to handle both REMINDER and SYMPTOM messages
- Display "checkout" instead of "symptom" in the LISA Communications UI

## Changes

### 1. Message template text updated

**Files:** `messageTemplate.ts`, `messageTemplate.spec.ts`

Updated `symptomTextMessageFromTemplate()` to return the new checkout message text with the HubSpot discharge instructions PDF link. Added test coverage for the template.

### 2. First-appointment guards removed

**File:** `webhookUpsertAppointments.ts`

Removed two guards that prevented checkout messages for patients who already had completed appointments:
- `otherCompletedAppts` query --- blocked if the patient had any other completed appointment
- `existingSymptom` query --- blocked if the patient already had any symptom message

Now every completed appointment triggers a checkout message. The existing dedup inside `setSymptomMessage` (findOne by `appointmentId + source`) prevents duplicates for the **same** appointment.

### 3. `dateToSend` anchored to appointment date (bug fix)

**File:** `webhookUpsertAppointments.ts`

**Before:** `setSymptomMessage(apptId, undefined, patientId)` --- `undefined` caused `getSendTimeAdjustedDate` to default to `new Date()`, meaning the message was scheduled 1 business day from when the **webhook fired**, not from when the appointment occurred. This meant the timing depended on when Looker happened to send the webhook, not the actual appointment date.

**After:** `setSymptomMessage(apptId, new Date(appt.start_date_time), patientId)` --- the message is now scheduled for 9 AM ET on the next business day after the **appointment date**.

Added `start_date_time` to the projection in the appointments query so the field is available.

**Example:**
| Scenario | Old behavior | New behavior |
|---|---|---|
| Appointment on Thursday, webhook fires Friday | SMS scheduled for Monday (1 biz day after Friday) | SMS scheduled for Friday (1 biz day after Thursday) |

### 4. Spruce archival generalized for SYMPTOM messages

**Files:** `spruce_conversation.ts`, `spruce_conversation.test.ts`

- Renamed `archiveReminderConversation` to `archiveSmsConversation`
- After the existing REMINDER lookup, added a second block that queries for SYMPTOM messages and runs the same internal-note + archive flow
- Reuses existing feature flags `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` and `SPRUCE_SMS_REMINDER_ARCHIVE` --- no new flags needed
- Added test cases for SYMPTOM archival (enabled, disabled, note failure scenarios)
- All existing REMINDER tests continue to pass unchanged

### 5. UI display labels

**Files:** `messageDTO.ts`, `MessageList.tsx`

- Added `MESSAGE_SOURCE_DISPLAY_LABELS` mapping in `messageDTO.ts` --- maps `MessageSource.SYMPTOM` to `"checkout"`, all others keep their current label
- Updated the Type column and filter dropdown in `MessageList.tsx` to use display labels instead of raw enum values
- The stored `source` value remains `"symptom"` (no MongoDB migration needed)

## Test plan

- [x] Unit tests: `messageTemplate.spec.ts` --- template returns new checkout text with HubSpot URL
- [x] Unit tests: `spruce_conversation.test.ts` --- SYMPTOM archival with flags enabled/disabled, note failure
- [x] Unit tests: `messageService.test.ts` --- all 45 tests pass (covers `setSymptomMessage`, `getSendTimeAdjustedDate`)
- [ ] Manual: POST webhook with completed appointment, verify Message doc created with correct `dateToSend` anchored to appointment date
- [ ] Manual: Verify second completed appointment for same patient also creates a message (guard removal)
- [ ] Manual: Trigger `sendMessageProcessing` job, verify SMS sent via Spruce
- [ ] Manual: Check Communications page shows "checkout" in Type column and filter dropdown
- [ ] Manual: Verify Spruce conversation archived after checkout SMS response (staging only)
