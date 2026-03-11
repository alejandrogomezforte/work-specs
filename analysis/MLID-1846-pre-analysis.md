# MLID-1846 — Pre-Analysis: Archive other texts after sending

**Analysis using Claude**

---

## Technical Breakdown

The system sends patients 3 types of SMS via Spruce: "what to expect" (pre-appointment), "appointment reminder" (day before), and "symptom follow-up" (post-appointment). Currently, **only reminder conversations get archived** in Spruce after sending. The operations team wants **all three message types archived** so their Spruce inbox doesn't pile up with outbound-only threads.

### What's already built

The archiving infrastructure **already exists** — it just only runs for REMINDER messages:

1. `archiveSpruceConversation(conversationId)` in `apps/web/services/integrations/spruce.ts` — calls Spruce API `updateConversation({ archived: true })`
2. `postSpruceInternalNote(conversationId, note)` — posts "Processed by LISA" note before archiving
3. `archiveReminderConversation()` in `apps/web/services/webhooks/handlers/spruce_conversation.ts` — the webhook handler that triggers archiving, but **only for `MessageSource.REMINDER`**

### Current flow (REMINDER only)

```
SMS sent via Spruce → Spruce webhook fires (conversationItem.created)
→ handleSpruceConversation() webhook handler
→ Updates Message doc with conversationId
→ archiveReminderConversation() checks:
    - Is this a REMINDER message? → YES → archive
    - Is this WHAT_TO_EXPECT or SYMPTOM? → IGNORED (no archiving)
```

### What needs to change

Extend the archiving logic in the Spruce webhook handler to also archive WHAT_TO_EXPECT and SYMPTOM conversations, not just REMINDER. The archiving infrastructure is already proven in production for REMINDER — same Spruce API call, no new flags needed.

### Approach

Broaden the existing `archiveReminderConversation()` function to also match `MessageSource.WHAT_TO_EXPECT` and `MessageSource.SYMPTOM`. Rename it to something like `archiveOutboundConversation()` to reflect its expanded scope.

### Key files to touch

| File | Change |
|------|--------|
| `apps/web/services/webhooks/handlers/spruce_conversation.ts` | Extend `archiveReminderConversation()` to handle WHAT_TO_EXPECT and SYMPTOM sources; rename function |

### How archiving works under the hood

1. Spruce sends a webhook when the outbound SMS creates a conversation
2. Webhook handler in `spruce_conversation.ts` gets the `conversationId` and `requestId`
3. It looks up the `Message` doc by `spruce.requestId` to determine the message source
4. If the source matches, it optionally posts an internal note, waits 2 seconds, then calls `archiveSpruceConversation(conversationId)`

### Scope

Small task — the archiving plumbing already exists. The main work is generalizing the filter in the webhook handler (currently hardcoded to REMINDER) to also cover the other two message sources.
