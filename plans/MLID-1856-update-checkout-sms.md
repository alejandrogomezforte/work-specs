# MLID-1856 --- Update Checkout SMS Timing, Text, and Archival

## Task Reference

- **Jira**: [MLID-1856](https://localinfusion.atlassian.net/browse/MLID-1856)
- **Story Points**: 5
- **Branch**: `feature/MLID-1856-update-checkout-sms`
- **Base Branch**: `develop`
- **Status**: To Do

---

## Summary

Update the "symptom" SMS message to function as a "checkout" message: new message text with updated discharge instructions URL, send after every completed appointment (not just the first), archive the Spruce conversation after sending (matching MLID-1696 reminder behavior), and display the type as "checkout" in the LISA communications table while keeping the stored enum value `'symptom'` unchanged.

---

## Codebase Analysis

Key findings from the investigation:

- **Message template**: `symptomTextMessageFromTemplate()` in `messageTemplate.ts` (line 58-60) is a zero-argument function returning a hardcoded string. Called by reference in `sendMessageJob.ts` line 341. No existing test coverage for this function.
- **Scheduling**: `setSymptomMessage()` in `messageService.ts` (lines 124-151) creates a Message doc with `dateToSend` set via `getSendTimeAdjustedDate(date, 1)` then `setHours(13, 0, 0, 0)`. The hour value 13 UTC = 9 AM EDT, which is the codebase convention for "9 AM Eastern". No timing change needed.
- **First-appointment guard**: `webhookUpsertAppointments.ts` lines 263-289 contain two guards: (1) check for other completed appointments, (2) check if patient already has ANY symptom message. Both must be removed so every completed appointment triggers a checkout message. The per-appointment dedup in `setSymptomMessage` (findOne by appointmentId + source) prevents duplicates.
- **Spruce archival**: `archiveReminderConversation()` in `spruce_conversation.ts` is hardcoded to `source: MessageSource.REMINDER`. It uses feature flags `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` and `SPRUCE_SMS_REMINDER_ARCHIVE`. The function must be generalized to also handle SYMPTOM messages.
- **UI display**: `MessageList.tsx` line 345 renders `{message.source}` directly as the Type column text. The filter dropdown (line 179-183) also uses raw enum values. A display label mapping is needed.
- **Enum**: `MessageSource.SYMPTOM = 'symptom'` in `messageDTO.ts` --- value must NOT change (stored in MongoDB).
- **Feature flags**: `SEND_SYMPTOM` already gates the send job. Reuse existing `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` and `SPRUCE_SMS_REMINDER_ARCHIVE` to gate checkout archival (same behavior as reminder archival, no new flags needed).

---

## Implementation Steps

### Step 1 --- Update message template text and add test

- **Files**:
  - `apps/web/services/jobs/definitions/appointmentReminder/messageTemplate.spec.ts`
  - `apps/web/services/jobs/definitions/appointmentReminder/messageTemplate.ts`
- **What**:
  - RED: Add a `describe('symptomTextMessageFromTemplate')` test block asserting the function returns the new message text (with the HubSpot discharge instructions URL).
  - GREEN: Update `symptomTextMessageFromTemplate()` to return the new text:
    ```
    Thank you for visiting Local Infusion! If you've experienced any symptoms, please do not hesitate to reach out for any questions and we will get back to you. For anything that is urgent, please reach out to your provider or 911. If not, please disregard. We have also provided some more information about discharge instructions in the link below. We hope you're doing well! :)\n\nhttps://6939519.fs1.hubspotusercontent-na1.net/hubfs/6939519/Order%20Set%20Forms%202022/Discharge%20Instructions.pdf
    ```
  - REFACTOR: None expected.

### Step 2 --- Remove first-appointment guards in webhook handler

- **Files**:
  - `apps/web/services/webhooks/handlers/util/webhookUpsertAppointments.ts`
- **What**:
  - Remove the `otherCompletedAppts` query (lines 264-273) and its guard (`if (otherCompletedAppts) continue`).
  - Remove the `existingSymptom` query (lines 280-284) and its guard (`if (existingSymptom) continue`).
  - After removal, every completed appointment that has a patient will call `setSymptomMessage()`. The existing dedup inside `setSymptomMessage` (findOne by `appointmentId + source`) prevents duplicate messages for the same appointment.
  - **Fix `dateToSend` anchor**: The old code passes `undefined` as the `date` arg to `setSymptomMessage()`, causing `getSendTimeAdjustedDate` to default to `new Date()` (the webhook execution time). This means the message is scheduled 1 business day from when the webhook fires, not from when the appointment occurred. To meet the requirement "send at 9 AM ET the next business day after the appointment":
    1. Add `start_date_time` to the projection on the appointments query (line 244): `{ projection: { _id: 1, we_infuse_patient_id: 1, we_infuse_id: 1, start_date_time: 1 } }`
    2. Pass the appointment date to `setSymptomMessage`: `setSymptomMessage(String(appt._id), new Date(appt.start_date_time), String(patient._id))`
    3. This anchors `getSendTimeAdjustedDate(appointmentDate, 1)` to the actual meeting date, so a Thursday appointment yields a Friday 9 AM ET send time regardless of when Looker fires the webhook.
  - Note: No test file exists for `webhookUpsertAppointments.ts`. Creating a full integration test for this file is out of scope for this task since the function uses raw MongoDB driver (`db.collection()`) extensively. The behavior change will be verified through manual testing. If desired, a follow-up task can add tests using a proper test database.

### ~~Step 3 --- Add feature flags for checkout SMS archival~~ (ELIMINATED)

> No new feature flags needed. Reuse existing `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` and `SPRUCE_SMS_REMINDER_ARCHIVE` to gate both REMINDER and SYMPTOM archival — same behavior, different message source.

### Step 4 --- Generalize Spruce archival to handle checkout (symptom) messages

- **Files**:
  - `apps/web/services/webhooks/handlers/spruce_conversation.test.ts`
  - `apps/web/services/webhooks/handlers/spruce_conversation.ts`
- **What**:
  - RED: Add new test cases in a new `describe` block `'handleSpruceConversation - SMS Checkout (Symptom) Archival'`:
    - `'should post internal note and archive for SYMPTOM messages when reminder flags enabled'` --- mock `Message.findOne` to return a SYMPTOM message when queried with `source: MessageSource.SYMPTOM`, enable `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` and `SPRUCE_SMS_REMINDER_ARCHIVE` flags.
    - `'should NOT archive SYMPTOM messages when reminder flags are disabled'`
    - `'should NOT archive SYMPTOM messages when note posting fails'`
  - GREEN: Refactor `archiveReminderConversation` to also handle SYMPTOM source:
    - Rename to `archiveSmsConversation(requestId, conversationId)`.
    - After the existing REMINDER lookup + archival logic, add a second block that:
      1. Queries `Message.findOne({ 'spruce.requestId': requestId, source: MessageSource.SYMPTOM })`.
      2. If found, runs the same internal-note + archive flow using the existing `FeatureFlag.SPRUCE_SMS_REMINDER_INTERNAL_NOTE` and `FeatureFlag.SPRUCE_SMS_REMINDER_ARCHIVE` flags.
    - Update the call site (line 257) to use the new function name.
  - REFACTOR: Extract the note-then-archive logic into a private helper `archiveConversationWithNote(conversationId, internalNoteFlag, archiveFlag, logPrefix)` to avoid duplicating the REMINDER and CHECKOUT flows.
  - Update existing tests: rename references from `archiveReminderConversation` to `archiveSmsConversation` if any test imports it directly (current tests only import `handleSpruceConversation`, so no change needed).

### Step 5 --- Add display label mapping for MessageSource in UI

- **Files**:
  - `apps/web/types/messageDTO.ts`
  - `apps/web/components/Modules/messages/MessageList.tsx`
- **What**:
  - In `messageDTO.ts`, add a `MESSAGE_SOURCE_DISPLAY_LABELS` constant:
    ```typescript
    export const MESSAGE_SOURCE_DISPLAY_LABELS: Record<MessageSource, string> = {
      [MessageSource.WHAT_TO_EXPECT]: 'what-to-expect',
      [MessageSource.SYMPTOM]: 'checkout',
      [MessageSource.REMINDER]: 'reminder',
      [MessageSource.REVERIFICATION]: 'reverification',
      [MessageSource.SKYRIZI_OBI_TRAINING]: 'skyrizi-obi-training',
      [MessageSource.SKYRIZI_COMPLETE_PROMOTION]: 'skyrizi-complete-promotion',
    };
    ```
    Only the SYMPTOM entry changes --- all others keep their current raw enum value as the display label.
  - In `MessageList.tsx`:
    - Import `MESSAGE_SOURCE_DISPLAY_LABELS`.
    - **Type column** (line 345): Replace `{message.source}` with `{MESSAGE_SOURCE_DISPLAY_LABELS[message.source]}`.
    - **Filter dropdown** (lines 179-183): Update the `<option>` text to use the display label while keeping the `value` as the raw enum value:
      ```tsx
      <option key={source} value={source}>
        {MESSAGE_SOURCE_DISPLAY_LABELS[source]}
      </option>
      ```
  - No test file exists for `MessageList.tsx`. Creating one is out of scope for this task but could be a follow-up.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/jobs/definitions/appointmentReminder/messageTemplate.ts` | Modify | Update `symptomTextMessageFromTemplate` return text |
| `apps/web/services/jobs/definitions/appointmentReminder/messageTemplate.spec.ts` | Modify | Add test for `symptomTextMessageFromTemplate` |
| `apps/web/services/webhooks/handlers/util/webhookUpsertAppointments.ts` | Modify | Remove first-appointment and existing-symptom guards |
| ~~`apps/web/utils/featureFlags/featureFlags.ts`~~ | ~~Modify~~ | ~~No longer needed — reusing existing `SPRUCE_SMS_REMINDER_*` flags~~ |
| `apps/web/services/webhooks/handlers/spruce_conversation.ts` | Modify | Generalize archival to handle SYMPTOM messages with new flags |
| `apps/web/services/webhooks/handlers/spruce_conversation.test.ts` | Modify | Add test cases for SYMPTOM archival flow |
| `apps/web/types/messageDTO.ts` | Modify | Add `MESSAGE_SOURCE_DISPLAY_LABELS` mapping |
| `apps/web/components/Modules/messages/MessageList.tsx` | Modify | Use display labels in Type column and filter dropdown |

---

## Testing Strategy

- **Unit tests**:
  - `messageTemplate.spec.ts`: Assert `symptomTextMessageFromTemplate()` returns the exact new message text including the HubSpot URL.
  - `spruce_conversation.test.ts`: New describe block for SYMPTOM archival:
    - Should archive when SYMPTOM message found and checkout flags enabled.
    - Should NOT archive when checkout flags disabled.
    - Should NOT archive when internal note posting fails.
    - Should handle archive failure gracefully.
    - Existing REMINDER tests must continue to pass unchanged.
- **Integration tests**: None --- the webhook handler uses raw MongoDB driver making proper integration tests complex. Deferred to follow-up.
- **Manual verification**:
  1. Trigger a completed appointment webhook for a patient who already has prior completed appointments. Verify a new SYMPTOM message is created (first-appointment guard removed).
  2. Verify the message text matches the new copy with the HubSpot URL.
  3. Verify the message `dateToSend` is set to 1 business day after completion at 9 AM Eastern (13:00 UTC).
  4. After the message sends, verify the Spruce conversation gets the "Processed by LISA" internal note and is archived (when feature flags are enabled).
  5. In LISA Communications page, verify the Type column shows "checkout" for SYMPTOM messages.
  6. Verify the Type filter dropdown shows "checkout" and filtering by it correctly filters SYMPTOM messages.

---

## Security Considerations

- **Input validation**: No new user input introduced. All changes are to backend scheduling logic and display labels.
- **Authorization**: No authorization changes. The Communications page access remains unchanged.
- **Feature flag gating**: Archival reuses existing `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` and `SPRUCE_SMS_REMINDER_ARCHIVE` flags — no new flags introduced. Both REMINDER and SYMPTOM archival are controlled by the same toggles.
- **PHI handling**: No PHI changes. The message text is a static template with no patient-specific data embedded.

---

## Open Questions

1. **Archival timing in production**: The existing archival has a 2-second `setTimeout` delay between posting the internal note and archiving. This is inherited from the REMINDER implementation. Is this delay sufficient for SYMPTOM messages, or should it be configurable?
2. **Existing patients with old symptom messages**: Patients who already received the old symptom text on their first appointment will now receive the new checkout text on subsequent completed appointments. Is this the intended behavior, or should there be any migration/cleanup of old messages?
3. ~~**Feature flag rollout order**~~: No longer applicable — reusing existing `SPRUCE_SMS_REMINDER_*` flags that are already deployed and toggled in production.
