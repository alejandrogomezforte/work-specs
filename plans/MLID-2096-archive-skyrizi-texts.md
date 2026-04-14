# MLID-2096 — Archive Skyrizi texts after sending

## Task Reference

- **Jira**: [MLID-2096](https://localinfusion.atlassian.net/browse/MLID-2096)
- **Story Points**: 1
- **Branch**: `feature/MLID-2096-archive-skyrizi-texts`
- **Base Branch**: `develop`
- **Status**: Done

---

## Summary

Add archival support for two Skyrizi SMS text types (`skyrizi-obi-training` and `skyrizi-complete-promotion`) so that their Spruce conversations are automatically archived after sending — matching the existing behavior for REMINDER, SYMPTOM, and WHAT_TO_EXPECT messages.

This is a direct extension of the work done in MLID-1846, which refactored the archival dispatcher into a config-driven loop. Adding new sources is now a one-liner in the config map.

---

## Codebase Analysis

Key findings from investigation:

- **`MessageSource` enum** (`apps/web/types/messageDTO.ts`): Already contains `SKYRIZI_OBI_TRAINING = 'skyrizi-obi-training'` and `SKYRIZI_COMPLETE_PROMOTION = 'skyrizi-complete-promotion'`
- **Skyrizi services** (`apps/web/services/messages/skyrizi/`): Already send messages with the correct source and set `spruce.requestId` after sending
- **`ARCHIVABLE_SOURCE_LOG_PREFIXES`** (`spruce_conversation.ts:108-112`): Config map that drives the archival dispatcher — currently has 3 entries (REMINDER, SYMPTOM, WHAT_TO_EXPECT)
- **`archiveSmsConversation`** (`spruce_conversation.ts:131-149`): Uses a single `$in` query with all keys from the config map — fully source-agnostic, no changes needed
- **`noteAndArchiveConversation`** (`spruce_conversation.ts:27-106`): Shared helper, unchanged — handles internal note + archive gated by feature flags
- **Test pattern** (`spruce_conversation.test.ts`): Each source has a `describe` block with 3 tests (happy path, flags disabled, note failure)

---

## Implementation Steps

### Step 1 — 🔴 RED: Write failing tests for both Skyrizi sources

- **File**: `apps/web/services/webhooks/handlers/spruce_conversation.test.ts`
- **What**: Add two new `describe` blocks mirroring the existing WHAT_TO_EXPECT pattern (lines 526-678):
  1. `describe('handleSpruceConversation - SMS Skyrizi OBI Training Archival')` — 3 tests
  2. `describe('handleSpruceConversation - SMS Skyrizi Complete Promotion Archival')` — 3 tests
- **Each describe block contains**:
  - `it('should post internal note and archive for [SOURCE] messages when reminder flags enabled')`
  - `it('should NOT archive [SOURCE] messages when reminder flags are disabled')`
  - `it('should NOT archive [SOURCE] messages when note posting fails')`
- **Mock setup**: `Message.findOne` returns a message with the corresponding `MessageSource` when the `$in` filter includes it
- **Expected**: Tests FAIL because the sources are not yet in `ARCHIVABLE_SOURCE_LOG_PREFIXES`

### Step 2 — 🟢 GREEN: Add Skyrizi sources to the config map

- **File**: `apps/web/services/webhooks/handlers/spruce_conversation.ts`
- **What**: Add 2 entries to `ARCHIVABLE_SOURCE_LOG_PREFIXES` (line 112):
  ```typescript
  [MessageSource.SKYRIZI_OBI_TRAINING]: 'SMS Skyrizi OBI Training Archival:',
  [MessageSource.SKYRIZI_COMPLETE_PROMOTION]: 'SMS Skyrizi Complete Promotion Archival:',
  ```
- **Also update the JSDoc** on `archiveSmsConversation` (line 117) to list all 5 sources instead of 3
- **Expected**: All tests PASS

### Step 3 — 🔵 REFACTOR: Verify and clean up

- **What**: Run full test suite, type check, lint, and format
- **Commands**:
  ```bash
  cd apps/web && npx jest services/webhooks/handlers/spruce_conversation.test.ts
  npm run types:check
  npm run lint
  npm run format
  ```
- **Verify**: No regressions in the existing 17+ spruce_conversation tests

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `apps/web/services/webhooks/handlers/spruce_conversation.ts` | Modify | Add 2 entries to `ARCHIVABLE_SOURCE_LOG_PREFIXES`, update JSDoc |
| `apps/web/services/webhooks/handlers/spruce_conversation.test.ts` | Modify | Add 2 new `describe` blocks (6 total test cases) |

---

## Testing Strategy

- **Unit tests**: 6 new test cases across 2 describe blocks, following the exact pattern of the existing WHAT_TO_EXPECT block
- **Manual verification**: Enable `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` + `SPRUCE_SMS_REMINDER_ARCHIVE` feature flags, trigger a Skyrizi OBI Training and Skyrizi Complete Promotion SMS, verify conversations are archived in Spruce

---

## Security Considerations

- **Input validation**: No new user input — archival is triggered by Spruce webhook callbacks
- **Authorization**: Webhook authentication middleware is already in place (unchanged)
- **PHI handling**: No PHI involved — archival only touches conversation metadata (conversationId, requestId)

---

## Notes: What Triggers Each Skyrizi Message

Unlike REMINDER or WHAT_TO_EXPECT (which are event-driven via appointment webhooks), both Skyrizi messages are **cron-scheduled jobs** that run automatically.

### Skyrizi OBI Training (`skyrizi-obi-training`)

- **Trigger**: Pulse cron job `skyriziObiTraining` — runs at `0 18 * * 1-5` (6:00 PM ET, Mon–Fri)
- **Feature flag**: `SEND_SKYRIZI_OBI_TRAINING` (defaults to `false`)
- **Target appointments**: **2nd appointment** on a Skyrizi order from the previous business day
  - `first_appt_on_order_series: false`
  - `first_listed_primary_med: 'Skyrizi'`
  - Status not cancelled
- **Eligibility checks**: Patient exists, not opted out of texts, valid phone, location has callback number, passes `APPOINTMENT_REMINDER_TARGET_CRITERIA` phone whitelist
- **Deduplication**: Skips patients who already received this message today (UTC)
- **Service**: `skyriziObiTrainingService.ts` → `skyriziBaseService.ts` → `messageService.ts` → Spruce API
- **Message content**: OBI training directions link (`https://abbv.ie/asge73`) + Skyrizi Complete hotline

### Skyrizi Complete Promotion (`skyrizi-complete-promotion`)

- **Trigger**: Pulse cron job `skyriziCompletePromotion` — runs at `0 18 * * 1-5` (same schedule)
- **Feature flag**: `SEND_SKYRIZI_COMPLETE_PROMOTION` (defaults to `false`)
- **Target appointments**: **1st appointment** on a Skyrizi order from the previous business day, **status must be "Complete"**
  - `first_appt_on_order_series: true`
  - `first_listed_primary_med: 'Skyrizi'`
  - `status: 'Complete'`
- **Eligibility checks**: Same as OBI Training
- **Service**: `skyriziCompletePromotionService.ts` → same base flow

### End-to-End Flow (Both Types)

```
Cron fires (6 PM ET weekdays)
  → Job handler checks feature flag
  → Queries appointments matching criteria for previous business day
  → Filters for eligible patients (phone, opt-out, location, whitelist)
  → Deduplicates (skip if already sent today)
  → Sends SMS via Spruce API → Message record created with spruce.requestId
  → Spruce delivers SMS and sends webhook callback to POST /api/webhooks/spruce_conversations
  → handleSpruceConversation() updates Message with conversationId
  → archiveSmsConversation() checks if source is in ARCHIVABLE_SOURCE_LOG_PREFIXES
  → noteAndArchiveConversation() posts internal note + archives conversation (if flags enabled)
```

**Key difference from REMINDER/WHAT_TO_EXPECT**: There's no appointment-creation webhook involved. The jobs query the DB directly on a schedule.

---

## Notes: Manual Testing Strategy

The archival code path is triggered by **Spruce's webhook callback**, not by the job itself. So the testing approach has two parts:

### Part 1 — Verify the archival logic (our code change)

This is what our unit tests cover. The change is only in `ARCHIVABLE_SOURCE_LOG_PREFIXES` — the archival function itself is unchanged and already proven for 3 other sources.

### Part 2 — End-to-end manual test

To fully test, you need a Message record with `source: 'skyrizi-obi-training'` (or `skyrizi-complete-promotion`) and then simulate the Spruce callback.

**Option A: Trigger the real job** (requires staging environment with Spruce integration)

1. Ensure feature flags are enabled in the admin panel:
   - `SEND_SKYRIZI_OBI_TRAINING` = true (or `SEND_SKYRIZI_COMPLETE_PROMOTION`)
   - `SPRUCE_SMS_REMINDER_INTERNAL_NOTE` = true
   - `SPRUCE_SMS_REMINDER_ARCHIVE` = true
2. Ensure `APPOINTMENT_REMINDER_TARGET_CRITERIA` allows your test phone number (or set to `*`)
3. Ensure there's an eligible Skyrizi appointment for the previous business day in the DB
4. Run the job manually (via worker process or Pulse admin)
5. Wait for Spruce to send the webhook callback → conversation should be archived

**Option B: Simulate the Spruce callback** (local dev, no real SMS needed)

1. Create a Message document in MongoDB with:
   ```json
   {
     "source": "skyrizi-obi-training",
     "type": "sms",
     "status": "sent",
     "spruce": { "requestId": "test-request-123" }
   }
   ```
2. POST to `http://localhost:8080/api/webhooks/spruce_conversations` with a payload simulating the Spruce `conversationItem.created` event:
   ```json
   {
     "id": "test-event-1",
     "type": "conversationItem.created",
     "eventTime": "2026-04-07T18:00:00Z",
     "data": {
       "object": {
         "id": "ci-test-456",
         "object": "conversationItem",
         "conversationId": "conv-test-789",
         "requestId": "test-request-123",
         "direction": "outbound",
         "appURL": "https://app.sprucehealth.com/list/conv-test-789",
         "text": "Test skyrizi message",
         "author": { "displayName": "LISA Bot" },
         "createdAt": "2026-04-07T18:00:00Z",
         "modifiedAt": "2026-04-07T18:00:00Z"
       }
     }
   }
   ```
   **Note**: This endpoint validates `x-spruce-signature` header using HMAC-SHA256, so you'll either need the signing secret from the Spruce integration settings, or temporarily bypass signature validation for local testing.
3. Check logs for `SMS Skyrizi OBI Training Archival: Processing conversation conv-test-789`
4. Verify the Spruce API was called to post internal note and archive

**Option B is the closest to what you've done before** (POST payload to webhook), except the trigger here is the Spruce callback webhook rather than an appointment-creation webhook.

---

## Open Questions

None — this is a straightforward config extension following an established pattern.
