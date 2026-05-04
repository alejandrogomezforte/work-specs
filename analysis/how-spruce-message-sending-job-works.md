# How the Spruce Message Sending Job Works

**Audience**: Engineers working on the Spruce SMS pipeline.
**Scope**: Outbound message lifecycle — from a `Message` document in `READY` state to a delivered SMS with a Spruce `requestId` persisted back to MongoDB.
**Out of scope**: Inbound webhooks, conversation archiving, the sweeper. See `docs/agomez/postmortem/unarchived-spruce-messages.md` for those.

---

## Part 1 — Simplified Timeline

### The Object

The job operates on documents in the `messages` MongoDB collection. Each document is a `Message`:

```
Message {
  _id:           ObjectId
  type:          'sms'
  source:        'reminder' | 'symptom' | 'what-to-expect' | ...
  status:        'ready' | 'sending' | 'sent' | 'error' | 'declined'
  dateToSend:    Date         // when this message should be sent
  immediately:   boolean      // if true, send on next job run regardless of dateToSend
  retry:         number       // incremented on failure (capped at MAX_RETRY = 5)
  appointmentId: ObjectId     // populated for reminder/symptom
  intakeId:      ObjectId     // populated for what-to-expect
  patientId:    ObjectId
  spruce: {
    requestId:      string    // returned by Spruce API after POST           ◄── only link to webhook
    txt:            string    // the SMS body that was sent
    phone:          string    // destination phone (E.164)
    conversationId: string    // set later by inbound Spruce webhook
    appUrl:         string    // set later by inbound Spruce webhook
    archivedAt:     Date      // set later by archive flow
  }
}
```

A message starts life with `status: ready` (created by `appointmentReminder`, `jotFormLoadSubmission`, or the `looker_appointments` webhook — see the postmortem for upstream creation). The `sendMessageJob` is the consumer: it picks up `ready` messages, calls Spruce, and transitions them to `sent` with `spruce.requestId` populated.

### The Process — One Job Run

```
┌─ Pulse scheduler fires `sendMessageProcessing` every 2 minutes (America/New_York)
│
├─ 1. INIT
│      Connect to MongoDB
│      Load feature flags: APPOINTMENT_REMINDER, SEND_SYMPTOM, SEND_WHAT_TO_EXPECT
│      Check holiday calendar (REMINDER messages still send on holidays)
│      Build the list of allowed `sources` for this run
│
├─ 2. QUERY
│      Find up to 30 messages where:
│        status = READY  AND  source ∈ allowed sources
│        AND (dateToSend within today's window  OR  immediately = true)
│      Sort by `immediately: -1` (send urgent first)
│
│      Find up to 20 additional messages where:
│        status = ERROR  AND  retry < 5
│      (retry pool — these are previous failures)
│
│      Filter out anything outside work hours unless `immediately: true`
│      Reschedule any non-REMINDER message that lands on a holiday → next business day
│
├─ 3. FAN OUT (Promise.allSettled)
│      For each surviving message — up to ~30 in parallel — do:
│
│        a) Set status = SENDING  →  message.save()
│        b) Look up patient + location (aggregate join)
│        c) Check opt-out / isAllowed → if blocked, throw (becomes DECLINED)
│        d) POST to Spruce API:
│              spruceApi.postMessageFromEndpoint({ destination, message }, { internalEndpointId })
│           Spruce delivers SMS, returns { requestId: 'asyncRequest_XXXX' }
│        e) Build update: { status: SENT, sentDate: now, patientId, spruce: { requestId, txt, phone } }
│        f) Object.assign onto message  →  message.save()    ◄── MLID-2167 immediate persist
│           (best-effort: failures here are logged, not thrown — SMS already sent)
│
├─ 4. RECONCILE (after all settled)
│      For each result:
│        - fulfilled:  re-apply update + save  (defensive deferred save)
│        - rejected:   set status = ERROR (or whatever err.status was), retry++, save
│
└─ 5. done(error)  →  Pulse marks the job complete
```

### Where the Race Condition Lives

Step 3d returns `requestId` synchronously, but step 3f `message.save()` is asynchronous. Spruce, meanwhile, fires its own outbound `conversationItem.created` webhook back to our system as soon as it accepts the message — and that webhook contains the same `requestId`. The webhook handler runs `Message.updateMany({ 'spruce.requestId': requestId })` to set `conversationId` on the message.

If Spruce's webhook arrives **before** step 3f's `save()` commits, the `updateMany` matches 0 documents and the message becomes orphaned: it has the SMS sent and a `requestId`, but never gets a `conversationId` and is never archived. With up to 30 messages racing in parallel, the gap between 3d and 3f stretches unpredictably.

This is the architectural problem. Everything below is the implementation that produces it.

---

## Part 2 — Deep Dive

### 2.1 Job registration and scheduling

Two job names are registered with Pulse, both wired to the same handler. This is a legacy artifact — `sendMessageJob` is the original, `sendMessageProcessing` is the one that's actually scheduled.

**File**: `apps/web/services/jobs/definitions/sendMessageJob/sendMessageJob.ts:471-491`

```typescript
export const defineSendMessageJob = async (): Promise<void> => {
  await defineJob('sendMessageJob', sendMessageJob, {
    attempts: 1,
    backoff: { type: 'fixed', delay: 1800000 },
    lockLifetime: 5 * 60 * 1000,
    priority: 'normal',
  });

  await defineJob('sendMessageProcessing', sendMessageJob, {
    attempts: 1,
    backoff: { type: 'fixed', delay: 1800000 },
    lockLifetime: 5 * 60 * 1000,
    priority: 'normal',
  });
};
```

`lockLifetime: 5 minutes` — Pulse holds an exclusive lock for that long so the same job doesn't run twice in parallel.

**Schedule** — `apps/web/services/jobs/scheduler.ts:729-740`:

```typescript
await scheduleRecurringJob(
  'sendMessageProcessing',
  '*/2 * * * *',                    // every 2 minutes
  {},
  {
    timezone: 'America/New_York',
    skipImmediate: true,
  }
);
```

So the job ticks every 2 minutes, but only sends when `isWorkHours(now)` is true (9 AM – 3 PM EST), per the filter in step 3.

### 2.2 Picking up messages — the two queries

The job runs **two** finds and concatenates them. The first is the main pool of `READY` messages, the second is a smaller retry pool of previously failed messages.

**File**: `sendMessageJob.ts:227-269`

```typescript
// Normal messages
messages = await Message.find({
  status: MessageStatus.READY,
  source: { $in: sources },
  $or: [
    { dateToSend: { $gte: start, $lte: end } },
    { immediately: true },
  ],
})
  .sort({ immediately: -1 })
  .limit(MAX_MESSAGES);            // MAX_MESSAGES = 30

// Failed messages — retry pool
errorMessages = await Message.find({
  status: MessageStatus.ERROR,
  retry: { $lt: MAX_RETRY },       // MAX_RETRY = 5
  $or: [
    { dateToSend: { $gte: start, $lte: end } },
    { immediately: true },
  ],
}).limit(MAX_ERROR_MESSAGES);      // MAX_ERROR_MESSAGES = 20
```

`getDateRange(now)` returns today's window in EST. `sort({ immediately: -1 })` makes urgent messages (`immediately: true`) bubble to the front of the batch.

After the merge:

```typescript
messages = [...messages, ...errorMessages].filter((message) => {
  return message.immediately || isWorkHours(now);
});
```

Messages outside work hours are dropped from this batch unless they're flagged `immediately`. Then a holiday rescheduler (`checkAndRescheduleForHoliday`) walks the list and pushes any non-REMINDER message that lands on a holiday to the next business day.

### 2.3 The fan-out

This is where the race condition is born. All messages are mapped through `Promise.allSettled` and run concurrently — there is no batching, no rate limiting, no backpressure. With `MAX_MESSAGES + MAX_ERROR_MESSAGES = 50` messages possible, you can have up to 50 in-flight HTTP calls and Mongoose saves at once.

**File**: `sendMessageJob.ts:309-409`

```typescript
const result: PromiseSettledResult<ResultStatus>[] =
  await Promise.allSettled(
    messages.map(async (message: IMessage) => {
      const item: ResultStatus = {
        entity: message,
        toUpdate: { status: MessageStatus.READY },
      };

      try {
        if (message.type === MessageType.SMS) {
          message.status = MessageStatus.SENDING;
          await message.save();                                    // (a) status flip

          if (message.source === MessageSource.WHAT_TO_EXPECT) {
            const { patient, requestId, txt, phone } =
              await sendSmsMessageByIntake({ ... });               // (b-d)
            item.toUpdate.status = MessageStatus.SENT;
            item.toUpdate.sentDate = new Date();
            item.toUpdate.patientId = patient._id as string;
            item.toUpdate.spruce = { requestId, txt, phone };
          } else if (message.source === MessageSource.SYMPTOM) {
            // ... same shape, sendSmsMessageByAppointment
          } else if (message.source === MessageSource.REMINDER) {
            // ... same shape, sendSmsMessageByAppointment + tracking callbacks
          }

          // MLID-2167: persist requestId immediately so the inbound webhook
          // can find this message. Best-effort — never re-throw here, because
          // the SMS has already been delivered.
          if (item.toUpdate.spruce) {
            Object.assign(message, item.toUpdate);
            try {
              await message.save();                                // (f) immediate persist
            } catch (saveErr) {
              logger.warn(
                'sendMessageJob: immediate spruce persist failed, deferred save will retry',
                { messageId: message._id, error: saveErr }
              );
            }
          }
        }
      } catch (err) {
        item.error = err;
        throw item as unknown as Error;                            // promotes to ERROR
      }

      return item;
    })
  );
```

Three things to notice:

1. **The status flip to `SENDING` is also a save.** That's a second round trip on top of the post-send save. With 50 concurrent messages, you have ~100 `message.save()` calls competing for I/O during a single run.
2. **The catch swallows-and-rethrows the whole `item` object** as a fake `Error`, which is then unpacked in the reconcile step. This is unusual and brittle.
3. **The post-send save is deliberately fire-and-log.** A failure there must not flip the status to ERROR, because the SMS already went out — re-sending would double-text the patient. The deferred save in step 4 is the fallback.

### 2.4 The send itself — `sendSmsMessageByAppointment` / `sendSmsMessageByIntake`

These two functions live in `apps/web/services/messages/messageService.ts`. They share the same shape: load context (appointment or intake → patient + location), check opt-out, then delegate to `sendSpruceSms`.

**File**: `messageService.ts:416-555` (excerpted)

```typescript
export const sendSmsMessageByAppointment = async ({ message, txt, sprApi, isAllowed, ... }) => {
  // Aggregate join: appointment → patient → location
  const appointment = await db.collection(COLLECTION.Appointments)
    .aggregate<IAppointment>([
      { $match: { _id: message.appointmentId } },
      { $lookup: { from: COLLECTION.Patients, localField: 'we_infuse_patient_id',
                   foreignField: 'we_infuse_id_IV', as: 'patient' } },
      { $addFields: { patient: { $arrayElemAt: ['$patient', 0] } } },
      { $lookup: { from: COLLECTION.PatientLocations, localField: 'we_infuse_location_id',
                   foreignField: 'we_infuse_id', as: 'location' } },
      { $addFields: { location: { $arrayElemAt: ['$location', 0] } } },
    ])
    .next();

  if (!appointment) throw new Error(...);

  // Resolve template if txt is a function
  if (typeof txt === 'function') {
    txt = txt({ dateTime: appointment.start_local_tz_time, address });
  }

  const { patient, location } = await getPatientLocationFromAppointment({ appointment, trackIssue });

  if (patient.text_message_opt_out || !isAllowed({ appointment, patient })) {
    const err = new Error(`... skip sending confirmation for member_id: ${patient.we_infuse_id_IV}`);
    (err as any).status = MessageStatus.DECLINED;       // ◄── tagged so reconcile uses DECLINED, not ERROR
    throw err;
  }

  const requestId = await sendSpruceSms(patient, location, txt, sprApi);

  return { patient, location, requestId, appointment, txt, phone: `+1${patient.mobile_phone}` };
};
```

The `(err as any).status = MessageStatus.DECLINED` trick is how the function signals "this isn't a real error, don't retry." The reconcile step in step 4 reads it back: `const status = item.reason?.error?.status ?? MessageStatus.ERROR`.

### 2.5 The Spruce HTTP call — `sendSpruceSms`

**File**: `messageService.ts:153-272`

Two HTTP calls happen here:

1. `spruceApi.internalEndpoints()` — GET, list our org's internal endpoints (phone numbers we own). Used to resolve `sourcePhoneNumber` (the location's `callback_number`) into Spruce's internal endpoint ID.
2. `sprApi.postMessageFromEndpoint(body, metadata)` — POST, the actual SMS send.

```typescript
export const sendSpruceSms = async (
  patient: IPatient,
  location: ILocation,
  txt: string,
  sprApi: typeof spruceApi
): Promise<string> => {
  const targetPhoneNumber = `+1${patient.mobile_phone}`;
  const sourcePhoneNumber = `+1${location.callback_number}`;

  // 1. Resolve internal endpoint ID
  const internalEndpointsResponse = await spruceApi.internalEndpoints();
  const internalEndpoint = internalEndpointsResponse.data.internalEndpoints.find(
    ({ endpoint }) =>
      endpoint.rawValue === sourcePhoneNumber &&
      ['phone', 'sms'].includes(endpoint.channel)
  );
  // (no internal endpoint found → warn, but continue with empty internalEndpointId)

  // 2. Send the SMS
  const endpointBody: PostMessageFromEndpointBody = {
    destination: { smsOrEmailEndpoint: targetPhoneNumber },
    message: { body: [{ type: 'text', value: txt }] },
  };
  const metadata = { internalEndpointId: internalEndpoint?.endpoint?.id ?? '' };

  const { data } = await sprApi.postMessageFromEndpoint(endpointBody, metadata);

  return data.requestId;                       // e.g., 'asyncRequest_2NE1B99RGG800'
};
```

**The important property of `requestId`**: it's the only correlation token between this synchronous response and the asynchronous `conversationItem.created` webhook that Spruce will fire moments later. Spruce does NOT include any of our identifiers (`messageId`, `appointmentId`, etc.) in either the response or the webhook payload. If we lose track of `requestId`, we have no way to link the webhook back to our message.

### 2.6 The Spruce client — `getSpruceApi` / `spruceApi`

`sprApi` is a generated client from Spruce's OpenAPI spec, auth-injected at module load.

**File**: `apps/web/services/integrations/spruce.ts:739-746`

```typescript
export async function getSpruceApi(): Promise<typeof spruceApi> {
  const integration = await getIntegration('spruce');
  if (!integration || !integration.enabled) {
    throw new Error('Spruce integration is not enabled');
  }
  await configureSpruceApi();           // pulls token from Azure Key Vault, caches
  return spruceApi;
}
```

`spruceApi` is imported from `@api/spruce` — the generated client lives in `apps/web/apis/spruce/`. The bearer token is fetched from Azure Key Vault (`spruce-api-token`) once and cached in the module via the `apiConfiguration` promise singleton.

### 2.7 The reconcile — deferred save

After `Promise.allSettled` returns, the job loops over the results and writes each one back to MongoDB. Fulfilled items get the SENT update applied (again — this is the redundant save that the MLID-2167 fix tried to make moot); rejected items get marked ERROR (or whatever `err.status` was tagged) with `retry++`.

**File**: `sendMessageJob.ts:411-442`

```typescript
await Promise.all(
  result.map(async (item: PromiseSettledResult<ResultStatus>) => {
    if (item.status === 'fulfilled') {
      Object.assign(item.value.entity, item.value.toUpdate);
      await item.value.entity.save();
    } else if (item.status === 'rejected') {
      const status = item.reason?.error?.status ?? MessageStatus.ERROR;

      Object.assign(item.reason.entity, item.reason.toUpdate ?? {}, {
        status,
        retry: (item.reason.entity.retry ?? 1) + 1,
        error: {
          message: item.reason?.error?.message ?? 'sendMessageJob: Unknown error',
        },
      });
      await item.reason.entity.save();
    }
  })
);
```

This step exists for two reasons: (a) before MLID-2167 it was the *only* place we persisted `requestId`; (b) it's still needed to handle rejections (since the inner `try` re-throws). Today it's a defensive double-save for fulfilled items.

### 2.8 The Message model — what gets persisted

**File**: `apps/web/models/Message.ts:46-65`

```typescript
spruce: {
  requestId:      { type: String },     // populated by step 3f or 4
  txt:            { type: String },
  phone:          { type: String },
  conversationId: { type: String },     // populated by INBOUND webhook (spruce_conversation.ts)
  appUrl:         { type: String },     // populated by INBOUND webhook
  archivedAt:     { type: Date },       // populated by archive flow / sweeper
}
```

There's an index on `'spruce.requestId'` (line 100), which is what the inbound webhook handler uses to look the message up. That index is populated by step 3f's save — so until 3f commits, the index doesn't have an entry for this `requestId`, and the webhook handler's lookup misses.

---

## Architectural Properties (For the Long-Term Discussion)

Worth naming explicitly for the redesign conversation:

1. **The job is a polling consumer with no backpressure.** It picks up to 50 messages per tick and fires them all at once. There is no concept of "we're already processing N, slow down."
2. **The Spruce send and the persist are two separate side effects with no transaction wrapping them.** There is no atomic "send + record." The MLID-2167 fix narrows the gap but doesn't close it.
3. **`requestId` is the only correlation token, and it only exists *after* the Spruce response.** Spruce does not let us pre-allocate or supply our own correlation ID in the request. So no design that hinges on "save before send" is possible without changing Spruce's API contract.
4. **The webhook is fundamentally faster than the local persist.** Spruce's webhook fires from Spruce's infrastructure as soon as they accept the message. Our `save()` requires waiting for the same TCP response we just got, then a round trip to MongoDB. The webhook can — and does — beat us.
5. **Two fail-safe layers exist** (sweeper Pass 1 for archive failures; sweeper Pass 2 for orphaned messages, MLID-2167), but they're reactive, not architectural. They mop up after the race instead of preventing it.

A long-term fix probably needs to either (a) make the persist happen *before* the webhook can possibly arrive (intent record + outbox pattern), or (b) make `Message.updateMany` retriable with a small delay so the webhook can wait for the DB to catch up. Both options are discussed separately.
