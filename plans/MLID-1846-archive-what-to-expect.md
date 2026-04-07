# MLID-1846 — Archive WHAT_TO_EXPECT conversations after sending

## Task Reference

- **Jira**: [MLID-1846](https://localinfusion.atlassian.net/browse/MLID-1846)
- **Story Points**: 1
- **Branch**: `feature/MLID-1846-archive-what-to-expect`
- **Base Branch**: `develop`
- **Status**: Done

---

## Summary

When WHAT_TO_EXPECT SMS messages are sent via Spruce, the resulting conversation threads are not archived, causing clutter. REMINDER and SYMPTOM messages already archive via `noteAndArchiveConversation`. This task adds the same behavior for WHAT_TO_EXPECT by extending the `archiveSmsConversation` dispatcher function and adding corresponding tests.

---

## Codebase Analysis

Key findings from the investigation:

- **Handler file**: `apps/web/services/webhooks/handlers/spruce_conversation.ts` contains two relevant functions:
  - `noteAndArchiveConversation(conversationId, logPrefix)` (lines 27-106) — shared helper that posts "Processed by LISA" internal note (gated by `SPRUCE_SMS_REMINDER_INTERNAL_NOTE`), waits 2s, then archives (gated by `SPRUCE_SMS_REMINDER_ARCHIVE`). This function is reused as-is.
  - `archiveSmsConversation(requestId, conversationId)` (lines 111-140) — dispatcher that queries `Message.findOne` for REMINDER and SYMPTOM sources. A third block for WHAT_TO_EXPECT must be appended here.
- **Enum value**: `MessageSource.WHAT_TO_EXPECT = 'what-to-expect'` already exists in `apps/web/types/messageDTO.ts`. No new enum value needed.
- **Feature flags**: Both `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` and `SPRUCE_SMS_REMINDER_ARCHIVE` already exist. No new flags needed.
- **Existing tests**: `apps/web/services/webhooks/handlers/spruce_conversation.test.ts` has two describe blocks — "SMS Reminder Archival" (9 tests) and "SMS Checkout (Symptom) Archival" (3 tests). The pattern is well-established: mock `Message.findOne` to return a message for the target source only, use `callOrder` array for sequence assertions, `jest.useFakeTimers` + `advanceTimersByTimeAsync(3000)` for the 2s delay.
- **Existing filter assertion test** (line 327-343): The test "should query Message.findOne with correct filters for REMINDER and SYMPTOM sources" asserts exactly two `findOne` calls. This test must be updated to also assert the WHAT_TO_EXPECT call.

---

## Implementation Steps

### Step 1 — RED: Write failing tests for WHAT_TO_EXPECT archival

- **Files**: `apps/web/services/webhooks/handlers/spruce_conversation.test.ts`
- **What**: Add a new `describe('handleSpruceConversation - SMS What To Expect Archival')` block following the same pattern as the Symptom block. Also update the existing filter assertion test in the Reminder block to expect WHAT_TO_EXPECT.

**New describe block** with 3 test cases (mirroring Symptom block):

1. `'should post internal note and archive for WHAT_TO_EXPECT messages when reminder flags enabled'`
   - Mock `Message.findOne` to return a message only for `MessageSource.WHAT_TO_EXPECT`
   - Assert `postSpruceInternalNote` called with `(mockConversationId, 'Processed by LISA')`
   - Assert `archiveSpruceConversation` called with `mockConversationId`

2. `'should NOT archive WHAT_TO_EXPECT messages when reminder flags are disabled'`
   - Mock both feature flags to return `false`
   - Assert neither `postSpruceInternalNote` nor `archiveSpruceConversation` called

3. `'should NOT archive WHAT_TO_EXPECT messages when note posting fails'`
   - Mock `postSpruceInternalNote` to reject
   - Assert `archiveSpruceConversation` not called

**Updated existing test** in the Reminder describe block:

- Test at line 327: `'should query Message.findOne with correct filters for REMINDER and SYMPTOM sources'` — rename to include WHAT_TO_EXPECT and add a third assertion:
  ```typescript
  expect(Message.findOne).toHaveBeenCalledWith({
    'spruce.requestId': mockRequestId,
    source: MessageSource.WHAT_TO_EXPECT,
  });
  ```

**Setup for new describe block** (same pattern as Symptom block):
- `mockConversationId = 'conv-what-to-expect-123'`
- `mockRequestId = 'asyncRequest_WHAT_TO_EXPECT_001'`
- `mockConversationItemId = 'ci-what-to-expect-456'`
- `Message.findOne` mock returns message only when `filter.source === MessageSource.WHAT_TO_EXPECT`
- `createOutboundWebhookPayload` helper with `id: 'webhook-event-3'` and `text: 'Your what to expect message'`
- Feature flags, `postSpruceInternalNote`, `archiveSpruceConversation` mocked identically to Symptom block

Run tests — all 3 new tests and the updated filter test should fail because implementation does not query for WHAT_TO_EXPECT yet.

### Step 2 — GREEN: Add WHAT_TO_EXPECT block to archiveSmsConversation

- **Files**: `apps/web/services/webhooks/handlers/spruce_conversation.ts`
- **What**: Append a third `Message.findOne` + `noteAndArchiveConversation` block after the SYMPTOM block in `archiveSmsConversation` (after line 139). Also update the JSDoc comment on line 108-109 to mention "what-to-expect".

Code to add after line 139:

```typescript
const whatToExpectMessage = await Message.findOne({
  'spruce.requestId': requestId,
  source: MessageSource.WHAT_TO_EXPECT,
});

if (whatToExpectMessage) {
  logger.info(
    `SMS What To Expect Archival: Processing conversation ${conversationId}`,
    { requestId, conversationId }
  );
  await noteAndArchiveConversation(
    conversationId,
    'SMS What To Expect Archival:'
  );
}
```

Run tests — all tests should pass.

### Step 3 — REFACTOR: Verify and clean up

- **Files**: Both files from Steps 1 and 2
- **What**: Run full lint, type check, and test suite. Confirm no regressions. No structural refactoring expected for a 1 SP task — the existing dispatcher pattern (sequential `findOne` calls) is appropriate for the current number of message sources (3).

Commands:
```bash
cd apps/web && npx jest spruce_conversation.test.ts
cd apps/web && npm run types:check
cd apps/web && npx eslint services/webhooks/handlers/spruce_conversation.ts services/webhooks/handlers/spruce_conversation.test.ts
```

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/webhooks/handlers/spruce_conversation.test.ts` | Modify | Add WHAT_TO_EXPECT describe block (3 tests), update filter assertion test to include WHAT_TO_EXPECT |
| `apps/web/services/webhooks/handlers/spruce_conversation.ts` | Modify | Add WHAT_TO_EXPECT findOne + noteAndArchiveConversation block in archiveSmsConversation, update JSDoc |

---

## Testing Strategy

- **Unit tests** (4 new/modified assertions):
  1. Happy path: WHAT_TO_EXPECT message found, both flags enabled — note posted and conversation archived
  2. Flags disabled: neither note nor archive called
  3. Note failure: archive not called when note posting throws
  4. Filter assertion: `Message.findOne` called with `{ source: MessageSource.WHAT_TO_EXPECT }` (updated existing test)
- **Manual verification**: Enable both feature flags in admin, trigger a WHAT_TO_EXPECT SMS send, observe in Spruce that the conversation gets the "Processed by LISA" note and is archived

---

## Security Considerations

- No new endpoints, no new input surfaces — this extends an existing internal webhook handler
- No new feature flags — reuses existing `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` and `SPRUCE_SMS_REMINDER_ARCHIVE`
- No PHI involved — conversation archival does not log or expose patient data

---

## How to Locally Test

### Unit tests (recommended)

```bash
cd apps/web && npx jest --testPathPattern="spruce_conversation.test" --no-coverage
```

All 17 tests cover the archival logic including the new WHAT_TO_EXPECT path.

### End-to-end (requires live Spruce integration)

1. Start the dev server: `npm run dev:web`

2. Enable feature flags in admin panel (or directly in MongoDB):
   - `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` → `true`
   - `SPRUCE_SMS_REMINDER_ARCHIVE` → `true`

3. Option A — **Trigger a real WHAT_TO_EXPECT SMS send** from the app (requires `SEND_WHAT_TO_EXPECT` flag enabled). When Spruce fires back the webhook, archival happens automatically. Check server logs for `SMS What To Expect Archival:` messages.

4. Option B — **Simulate the Spruce webhook** by POSTing to `http://localhost:8080/api/webhooks/spruce/conversation`:

   Prerequisites:
   - A `Message` doc in MongoDB with `source: 'what-to-expect'` and `spruce.requestId: '<some-request-id>'`

   Payload:
   ```json
   {
     "id": "test-event",
     "type": "conversationItem.created",
     "data": {
       "object": {
         "id": "ci-test",
         "conversationId": "<spruce-conversation-id>",
         "direction": "outbound",
         "requestId": "<some-request-id>",
         "appURL": "https://app.sprucehealth.com/list/test",
         "attachments": [],
         "author": { "displayName": "LISA" },
         "text": "test"
       }
     }
   }
   ```

5. Verify in server logs that you see:
   - `SMS What To Expect Archival: [STEP 1/2] Posting "Processed by LISA" note...`
   - `SMS What To Expect Archival: [STEP 2/2] Archiving SMS conversation...`

> **Note:** The webhook flow requires a live Spruce integration and valid conversation IDs. Staging/training environment with Spruce connected is the best place for full E2E verification.

---

## Open Questions

None — the pattern is established by REMINDER and SYMPTOM, and all required enum values and feature flags already exist.
