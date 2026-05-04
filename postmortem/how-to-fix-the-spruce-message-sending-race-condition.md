# Plan: Fix the Spruce Message Sending Race Condition

**Status**: Proposal
**Branch (proposed)**: `fix/MLID-XXXX-serialize-spruce-sends`
**Related**:
- `docs/agomez/postmortem/unarchived-spruce-messages.md`
- `docs/agomez/analysis/how-spruce-message-sending-job-works.md`
- `docs/agomez/plans/MLID-2167-requestid-recovery-plan.md`

---

## Goal

Eliminate orphaned Spruce SMS messages caused by the race condition in `sendMessageJob`. After this change, when Spruce fires its outbound `conversationItem.created` webhook, the corresponding `Message` document MUST already contain `spruce.requestId` in MongoDB.

The goal is **practical zero** errors. Mathematical zero is impossible at this layer — see "What this fix does NOT solve" below.

---

## Background

The MLID-2167 fix moved the `requestId` save to happen immediately after each individual Spruce send (line 386-399 in `sendMessageJob.ts`). It narrowed the race window but did not eliminate it. Orphans still occur when sending a bulk batch.

Root cause is **contention**, not the save itself. One job tick today does:

- Up to 50 messages in flight via `Promise.allSettled` (30 READY + 20 retry pool)
- Each message does 2 saves (`SENDING` flip + post-send persist) = up to ~100 concurrent Mongoose operations
- All competing for: MongoDB connection pool, Node.js event loop, TCP sockets, Mongoose middleware
- Plus a third deferred save per message in the reconcile phase after `Promise.allSettled` settles

Under that load, an individual `message.save()` can stall hundreds of ms — long enough for Spruce's webhook to win.

---

## The Fix

Two changes, both confined to `sendMessageJob.ts`. No changes to webhook handler, sweeper, or model.

### Change 1: Serialize message processing

Replace the `Promise.allSettled(messages.map(...))` fan-out with a sequential `for...of` loop. Each message completes its full lifecycle (Spruce POST + persist) before the next message begins.

### Change 2: Use majority writeConcern + journaling on the critical save

After the Spruce POST returns, persist `requestId` with `{ w: 'majority', j: true }`. This guarantees the write is durable on a majority of replicas AND flushed to the on-disk journal before the await returns. Eliminates the secondary failure mode where the webhook handler reads a stale replica.

### Bonus cleanup

The post-`allSettled` deferred reconcile phase (lines 411-442) becomes redundant once the loop body handles both fulfilled and rejected outcomes inline. Delete it.

---

## Why This Reaches Practical Zero

After both changes, the timeline for any single message becomes:

```
t=0      ── postMessageFromEndpoint returns with requestId
t=0+1ms  ── message.save() begins (no contention; dedicated event loop turn)
t=0+5ms  ── majority + journal write returns; document is globally visible
         ── loop moves to next message
```

For Spruce's webhook to win, their dispatcher would have to fire the webhook AND have it traverse the public internet AND get processed by our handler in less than ~5 ms. Real-world webhook delivery is tens to hundreds of ms. The race window closes faster than any webhook can fly.

---

## What This Fix Does NOT Solve

Spruce returns `requestId` only **after** they've already accepted the SMS. At that exact instant their webhook is queued on their side. We cannot have the `requestId` in our DB before they could possibly fire — the ordering is physically determined.

So the fix relies on the latency between "Spruce accepts + queues webhook" and "webhook arrives at our handler" being larger than the latency of our `save()`. In practice it is, by orders of magnitude. But it is not a mathematical proof.

For absolute zero, we would need either:
- An idempotency / correlation key that Spruce accepts pre-flight (would require Spruce API changes)
- A retry strategy on the webhook handler when `Message.updateMany` returns `modifiedCount: 0` (out of scope here; see MLID-2167 plan)
- An outbox-style pre-write before the POST (requires schema or workflow change)

This plan is the highest-value fix that fits inside the sending process alone.

---

## Trade-offs

| Aspect | Current (parallel) | After (serial) |
|---|---|---|
| Throughput per tick | ~3 s for 30 messages | ~15 s for 30 messages |
| Saves per message | 3 (SENDING flip + post-send + deferred) | 2 (SENDING flip + post-send) |
| Connection pool load | Up to ~100 concurrent ops | 1-2 ops at a time |
| Event loop pressure | High during fan-out | Minimal |
| Race window per message | 50–500 ms (under contention) | 1–5 ms |
| Predictability | Variable | Consistent |

The throughput cost is acceptable: the job runs every 2 minutes during work hours (9 AM – 3 PM EST), and `MAX_MESSAGES + MAX_ERROR_MESSAGES = 50`. A 15 s tick fits easily inside the 2-minute interval and well within the 5-minute `lockLifetime`.

If 15 s ever becomes a problem, lower `MAX_MESSAGES` to spread the work across more ticks rather than reintroducing parallelism.

---

## Implementation

### Files to Modify

| File | Change |
|---|---|
| `apps/web/services/jobs/definitions/sendMessageJob/sendMessageJob.ts` | Replace `Promise.allSettled` fan-out with serial loop; persist with majority writeConcern; delete deferred reconcile phase |
| `apps/web/services/jobs/definitions/sendMessageJob/__tests__/sendMessageJob.test.ts` | Update tests to assert serial ordering and writeConcern |

### Files NOT to Modify

| File | Reason |
|---|---|
| `services/messages/messageService.ts` | Send functions already correct; only the orchestration changes |
| `services/webhooks/handlers/spruce_conversation.ts` | Out of scope — fix lives in the sender |
| `services/jobs/definitions/spruceArchiveSweeper.ts` | Stays as the safety net for true edge cases |
| `models/Message.ts` | Schema unchanged |
| `services/jobs/scheduler.ts` | Schedule unchanged (still every 2 min) |

### Refactored Code Sketch

Replace lines 309-442 of `sendMessageJob.ts` with:

```typescript
const spruceApi = await getSpruceApi();

for (const message of messages) {
  try {
    if (message.type !== MessageType.SMS) continue;

    // 1. Flip status to SENDING
    message.status = MessageStatus.SENDING;
    await message.save();

    // 2. Send via Spruce — returns requestId
    const sendResult = await sendBySource(message, spruceApi);
    if (!sendResult) continue; // source not handled

    // 3. Persist requestId with strong durability BEFORE moving on.
    //    This is the critical save that closes the race window.
    Object.assign(message, {
      status: MessageStatus.SENT,
      sentDate: new Date(),
      patientId: sendResult.patient._id,
      spruce: {
        requestId: sendResult.requestId,
        txt: sendResult.txt,
        phone: sendResult.phone,
      },
    });
    await message.save({ writeConcern: { w: 'majority', j: true } });

    logger.info(
      `sendMessageJob: message source '${message.source}' sent (${message._id})`
    );
  } catch (err) {
    const status = (err as any)?.status ?? MessageStatus.ERROR;
    Object.assign(message, {
      status,
      retry: (message.retry ?? 1) + 1,
      error: {
        message: (err as Error)?.message ?? 'sendMessageJob: Unknown error',
      },
    });
    await message.save();

    logger.error(
      `sendMessageJob: message source '${message.source}' failed (${message._id})`
    );
  }
}
```

Where `sendBySource` is a small helper that dispatches to one of the existing message-service functions:

```typescript
const sendBySource = async (
  message: IMessage,
  spruceApi: typeof spruceApiType
): Promise<SendResult | null> => {
  switch (message.source) {
    case MessageSource.WHAT_TO_EXPECT:
      return sendSmsMessageByIntake({
        message,
        txt: whatToExpectMessageFromTemplate,
        sprApi: spruceApi,
        isAllowed: ({ patient }) => isAllowedToSendConfirmation(patient),
      });
    case MessageSource.SYMPTOM:
      return sendSmsMessageByAppointment({
        message,
        txt: symptomTextMessageFromTemplate,
        sprApi: spruceApi,
        isAllowed: ({ patient }) => isAllowedToSendConfirmation(patient),
      });
    case MessageSource.REMINDER:
      return sendSmsMessageByAppointment({
        message,
        txt: messageFromTemplate,
        sprApi: spruceApi,
        isAllowed: ({ patient, appointment }) =>
          isAllowedToSend(patient, appointment),
        trackIssue: trackReminderIssue,
        trackSent: trackReminderSent,
      });
    default:
      return null;
  }
};
```

This is also a readability win: the three near-identical `if/else` branches collapse into one switch.

---

## TDD Plan

Follow strict red-green-refactor per `CLAUDE.md`. The existing test file (`sendMessageJob.test.ts`) already covers most behaviors — the new tests below are additions or modifications.

### RED tests to write first

1. **`should process messages serially, not in parallel`**
   Mock `sendSmsMessageByAppointment` so each call resolves on a controlled timer. Assert that the second call begins only after the first has fully completed (including its post-send save).

2. **`should persist requestId with majority writeConcern`**
   Spy on `message.save`. Assert the post-send save is called with `{ writeConcern: { w: 'majority', j: true } }`.

3. **`should call save() exactly twice per fulfilled message (SENDING flip + final persist)`**
   Verify the deferred reconcile save is gone. With current code each fulfilled message triggers 3 saves; after the fix it should be 2.

4. **`should still mark message as ERROR and increment retry on send failure`**
   Force `sendSmsMessageByAppointment` to throw. Assert the message ends up with `status: ERROR`, `retry: prev + 1`, and `error.message` populated. This exercises the catch branch of the new loop.

5. **`should respect the DECLINED status tag from opt-out errors`**
   Throw an error with `(err as any).status = MessageStatus.DECLINED`. Assert the message saves as `DECLINED`, not `ERROR`. (Preserves existing semantics from the rejection path.)

6. **`should continue processing the next message when one fails`**
   In a batch of 3, force the middle one to throw. Assert messages 1 and 3 still succeed.

### GREEN — implementation

Refactor `sendMessageJob` per the sketch above. The existing `Promise.allSettled` block and the deferred reconcile block both go away.

### REFACTOR

- Extract `sendBySource` into the same file (or a sibling module if it grows).
- Confirm all existing tests still pass.
- Run `npm run lint:fix` and `npm run types:check` from `apps/web/`.
- Confirm test coverage on `sendMessageJob.ts` stays ≥ 80%.

---

## Verification

### Automated

1. `npm run test -- sendMessageJob` from `apps/web/`
2. `npm run types:check` from `apps/web/`
3. `npm run lint:fix` from `apps/web/`

### Manual (post-deploy)

1. Watch `messages` collection for 24-48 hours after rollout.
2. Run an aggregation to count orphans created since deploy:
   ```javascript
   db.messages.countDocuments({
     createdAt: { $gte: ISODate('<deploy-date>') },
     'spruce.requestId': { $exists: true, $ne: '' },
     'spruce.conversationId': { $exists: false },
     status: 'sent',
   })
   ```
   Expected: 0 (or very close — the sweeper will catch any genuine outliers).
3. Confirm the existing sweeper Pass 2 logs no "orphan recovery" attempts for messages sent after the deploy.

### Manual (job behavior)

1. Trigger `sendMessageProcessing` from `/admin/jobs` with a backlog of 30+ READY messages.
2. Confirm logs show messages processed one at a time (not interleaved).
3. Confirm total tick duration is reasonable (~10-20 s for 30 messages).
4. Confirm no `lockLifetime` exceeded warnings.

---

## Rollout

Low risk — the change is a pure refactor of behavior already proven correct, just with different concurrency semantics. No schema, no API, no webhook, no client.

1. Land in `develop`.
2. Deploy to staging.
3. Trigger a sizeable batch send manually via admin UI.
4. Confirm no orphans created in staging.
5. Promote to production.
6. Monitor `messages` collection for 1 week. Sweeper Pass 2 recovery count should drop to ~0.

If a regression appears (e.g., the job consistently runs longer than expected), revert is one PR — the old code remains in git history.

---

## Future Work (Out of Scope)

Once this is in place, the sweeper Pass 2 should rarely fire. If it never fires for 30+ days, consider:

- Demoting it to a daily run (instead of every 15 min).
- Adding a dashboard metric that alerts if Pass 2 recovers more than N messages per day — that becomes the canary that tells us the race is back.

A more ambitious follow-up would be the **outbox pattern**: write a "send intent" record before the Spruce POST, with a client-generated correlation ID, and have the webhook handler match on that. That eliminates the race mathematically but requires changing how the webhook handler does lookups. Not recommended unless this fix proves insufficient in production.
