# Postmortem Follow-up: Spruce Archival Fix Verification (MLID-2167)

**Date**: 2026-04-23
**Branch deployed**: `fix/MLID-2167-spruce-archive-requestid-recovery`
**Companion document**: `unarchived-spruce-messages.md` (original incident)

---

## Purpose

Validate end-to-end that the MLID-2167 fix prevents Spruce SMS messages from becoming permanently orphaned. The fix consists of two parts:

1. **Fix 1 — `spruce_conversation.ts`**: Use dot-notation `$set` on the `.updated` webhook so the empty `requestId` no longer overwrites the value saved by the `.created` webhook.
2. **Fix 2 — `spruceArchiveSweeper.ts`**: Pass 2 in the sweeper recovers orphaned messages by looking up `requestId` in `conversationitems` and backfilling `conversationId` + `appUrl` on the `messages` document.

Goal: send a controlled batch of SMS messages and confirm 100% are archived (`spruce.conversationId`, `spruce.appUrl`, `spruce.archivedAt` all populated).

---

## Test Setup

- **Patient**: existing test patient (`we_infuse_patient_id: 491052`)
- **Location**: `1229`
- **Phone**: redacted
- **Appointment date**: `2026-04-24 10:30:00` (2 days out — within reminder window, but `status: Complete` triggers the **symptom** message path instead)
- **Appointment range**: `we_infuse_id` 99999951 → 99999960 (10 demo appointments)
- **Spruce conversation**: all 10 messages route to thread `t_2GUJ2OR71KG00` (same patient/phone)

---

## Steps

### Step 1 — POST 10 demo appointments to the `looker_appointments` webhook

Single POST body with 10 entries in `data[]`, each identical except for `appointments.id` (99999951–99999960). All with `status: Complete` so `webhookUpsertAppointments` triggers `setSymptomMessage`.

```js
db.appointments.find({ we_infuse_id: { $gte: 99999951, $lte: 99999960 } })
```

→ 10 appointments created.

| we_infuse_id | Mongo _id |
|---|---|
| 99999951 | `69e963188e990a7c3100da45` |
| 99999952 | `69e963188e990a7c3100da46` |
| 99999953 | `69e963188e990a7c3100da47` |
| 99999954 | `69e963188e990a7c3100da48` |
| 99999955 | `69e963188e990a7c3100da49` |
| 99999956 | `69e963188e990a7c3100da4a` |
| 99999957 | `69e963188e990a7c3100da4b` |
| 99999958 | `69e963188e990a7c3100da4c` |
| 99999959 | `69e963188e990a7c3100da4d` |
| 99999960 | `69e963188e990a7c3100da4e` |

### Step 2 — Retrieve the messages auto-created for each appointment

`setSymptomMessage` (called from `webhookUpsertAppointments` on `Complete` status) inserted one `Message` per appointment with `source: symptom`, `status: READY`, `dateToSend: next-day 1 PM EST`.

```js
db.messages.find({
  appointmentId: { $in: [
    ObjectId("69e963188e990a7c3100da45"),
    ObjectId("69e963188e990a7c3100da46"),
    ObjectId("69e963188e990a7c3100da47"),
    ObjectId("69e963188e990a7c3100da48"),
    ObjectId("69e963188e990a7c3100da49"),
    ObjectId("69e963188e990a7c3100da4a"),
    ObjectId("69e963188e990a7c3100da4b"),
    ObjectId("69e963188e990a7c3100da4c"),
    ObjectId("69e963188e990a7c3100da4d"),
    ObjectId("69e963188e990a7c3100da4e")
  ] }
})
```

→ 10 messages, one per appointment (see mapping table at the bottom).

### Step 3 — Force immediate sending (bypass work-hours window)

Test was run at ~17:41 EDT, outside the 9 AM – 3 PM EDT `sendMessageJob` window. Setting `immediately: true` bypasses both the `dateToSend` filter and the `isWorkHours()` gate (`sendMessageJob.ts:277`).

```js
db.messages.updateMany(
  { _id: { $in: [
    ObjectId("69e96318388f08020cd9ca81"),
    ObjectId("69e96318388f08020cd9ca87"),
    ObjectId("69e96318388f08020cd9ca8d"),
    ObjectId("69e96318388f08020cd9ca93"),
    ObjectId("69e96318388f08020cd9ca99"),
    ObjectId("69e96319388f08020cd9ca9f"),
    ObjectId("69e96319388f08020cd9caa5"),
    ObjectId("69e96319388f08020cd9caab"),
    ObjectId("69e96319388f08020cd9cab1"),
    ObjectId("69e96319388f08020cd9cab7")
  ] } },
  { $set: { immediately: true, dateToSend: new Date() } }
)
```

### Step 4 — Wait for `sendMessageJob` to fire

The job picked up all 10 within seconds, called Spruce, persisted `spruce.requestId` immediately (the MLID-2167 commit `a5d014cb` change), and marked each as `status: sent`.

| Event | Time (UTC) |
|---|---|
| Messages sent (range) | `00:28:02.203` – `00:28:03.194` |
| Messages archived (range) | `00:28:04.303` – `00:28:06.385` |
| Max sent → archived gap | ~3.4 s |

### Step 5 — Verify the `messages` collection has full Spruce metadata

```js
db.messages.find({
  _id: { $in: [
    ObjectId("69e96318388f08020cd9ca81"),
    ObjectId("69e96318388f08020cd9ca87"),
    ObjectId("69e96318388f08020cd9ca8d"),
    ObjectId("69e96318388f08020cd9ca93"),
    ObjectId("69e96318388f08020cd9ca99"),
    ObjectId("69e96319388f08020cd9ca9f"),
    ObjectId("69e96319388f08020cd9caa5"),
    ObjectId("69e96319388f08020cd9caab"),
    ObjectId("69e96319388f08020cd9cab1"),
    ObjectId("69e96319388f08020cd9cab7")
  ] }
})
```

All 10 documents had:
- `status: "sent"`
- `spruce.requestId`: present (`asyncRequest_*`)
- `spruce.conversationId`: `t_2GUJ2OR71KG00` (all share same Spruce thread)
- `spruce.appUrl`: present
- `spruce.archivedAt`: present

### Step 6 — Verify the `conversationitems` collection retains `requestId`

> Collection name is `conversationitems` (lowercase) — Mongoose lowercases the model name when creating the collection. The `COLLECTION.ConversationItems` constant value (`'conversationItems'`) is the model name, not the collection.

```js
db.conversationitems.find({
  "data.data.object.requestId": { $in: [
    "asyncRequest_2NEAUT0L07000",
    "asyncRequest_2NEAUSVVOD000",
    "asyncRequest_2NEAUT7CO7000",
    "asyncRequest_2NEAUT2QGD000",
    "asyncRequest_2NEAUT5RO7000",
    "asyncRequest_2NEAUT2BG7000",
    "asyncRequest_2NEAUT5AO7000",
    "asyncRequest_2NEAUT778D000",
    "asyncRequest_2NEAUT628D000",
    "asyncRequest_2NEAUT7O87000"
  ] }
})
```

→ 10 documents found, all type `conversationItem.created`, all with non-empty `requestId`. **Fix 1 verified at the `.created` step** — none of the items had `requestId` blanked out.

---

## Results

**100% archival success.** All 10 messages reached the fully-archived state with no orphans.

| Message _id | Appointment _id | requestId | conversationItem (`ti_`) | sent (UTC) | archived (UTC) | spruce.conversationId | archived |
|---|---|---|---|---|---|---|---|
| `69e96318388f08020cd9ca81` | `69e963188e990a7c3100da45` | `asyncRequest_2NEAUT0L07000` | `ti_2NEAUT2E9RG00` | `00:28:02.314` | `00:28:05.151` | `t_2GUJ2OR71KG00` | ✅ |
| `69e96318388f08020cd9ca87` | `69e963188e990a7c3100da46` | `asyncRequest_2NEAUSVVOD000` | `ti_2NEAUT1HS6O00` | `00:28:02.203` | `00:28:04.303` | `t_2GUJ2OR71KG00` | ✅ |
| `69e96318388f08020cd9ca8d` | `69e963188e990a7c3100da47` | `asyncRequest_2NEAUT7CO7000` | `ti_2NEAUTA1S6O00` | `00:28:03.148` | `00:28:06.359` | `t_2GUJ2OR71KG00` | ✅ |
| `69e96318388f08020cd9ca93` | `69e963188e990a7c3100da48` | `asyncRequest_2NEAUT2QGD000` | `ti_2NEAUT4QA0000` | `00:28:02.563` | `00:28:05.241` | `t_2GUJ2OR71KG00` | ✅ |
| `69e96318388f08020cd9ca99` | `69e963188e990a7c3100da49` | `asyncRequest_2NEAUT5RO7000` | `ti_2NEAUTAKS6O00` | `00:28:02.951` | `00:28:06.281` | `t_2GUJ2OR71KG00` | ✅ |
| `69e96319388f08020cd9ca9f` | `69e963188e990a7c3100da4a` | `asyncRequest_2NEAUT2BG7000` | `ti_2NEAUT56HRG00` | `00:28:02.509` | `00:28:06.370` | `t_2GUJ2OR71KG00` | ✅ |
| `69e96319388f08020cd9caa5` | `69e963188e990a7c3100da4b` | `asyncRequest_2NEAUT5AO7000` | `ti_2NEAUTB61RG00` | `00:28:02.886` | `00:28:06.280` | `t_2GUJ2OR71KG00` | ✅ |
| `69e96319388f08020cd9caab` | `69e963188e990a7c3100da4c` | `asyncRequest_2NEAUT778D000` | `ti_2NEAUT9LC6O00` | `00:28:03.125` | `00:28:06.342` | `t_2GUJ2OR71KG00` | ✅ |
| `69e96319388f08020cd9cab1` | `69e963188e990a7c3100da4d` | `asyncRequest_2NEAUT628D000` | `ti_2NEAUT8RS6O00` | `00:28:02.977` | `00:28:06.385` | `t_2GUJ2OR71KG00` | ✅ |
| `69e96319388f08020cd9cab7` | `69e963188e990a7c3100da4e` | `asyncRequest_2NEAUT7O87000` | `ti_2NEAUT949RG00` | `00:28:03.194` | `00:28:06.177` | `t_2GUJ2OR71KG00` | ✅ |

---

## Findings

### Happy path is healthy

For all 10 messages:
- The Spruce `conversationItem.created` webhook arrived **after** the message's `requestId` was persisted to MongoDB → `Message.updateMany({ 'spruce.requestId': requestId })` matched, set `conversationId` + `appUrl`.
- `archiveSmsConversation` ran inline, posted the "Processed by LISA" note, and stamped `spruce.archivedAt`.
- End-to-end (sent → archived) completed in **< 4 seconds** for every message.

### Fix 1 status

Validated **for the `.created` path**: every conversationitem has a non-empty `requestId`. The `.updated` webhook had not yet fired for these items at query time (`createdAt == updatedAt` on every doc). Pre-fix, ~86% of outbound items in production had `requestId == ""` because of the `$set: { data }` overwrite. Post-fix sample: 0 / 10 lost the `requestId`.

To fully exercise Fix 1, re-query the same conversationitems after Spruce sends the `.updated` webhook (typically minutes to hours later) and confirm `requestId` is still non-empty.

### Fix 2 status

Pass 2 of `spruceArchiveSweeper` was **not exercised by this test** — none of the 10 messages were orphaned, so the sweeper had nothing to recover. The original failure mode (race condition: webhook arrives before `requestId` is persisted) is non-deterministic and was not reproduced in this run.

To validate Fix 2 directly, would need either:
- An artificially-induced delay in `sendMessageJob` between Spruce response and `message.save()`, or
- A retroactive scan over historical orphaned messages (those with `spruce.requestId` and no `spruce.conversationId`) once the sweeper has been running for one full lookback window.

---

## What we did NOT verify (open work)

1. **`.updated` webhook doesn't blank `requestId`** — needs a follow-up query after Spruce fires the `.updated` webhook for these items.
2. **Pass 2 sweeper recovers a real orphan** — needs either an injected race condition or a wait-and-scan against historical orphans.
3. **End-to-end flow under genuine load** — this test was 10 messages from a single patient. Production peaks are higher.

---

## Reference data

All raw test data (payload, IDs, Compass filters) is preserved in `docs/agomez/tmp.md`.

Filters used (Compass):

```js
// Appointments
{ we_infuse_id: { $gte: 99999951, $lte: 99999960 } }

// Messages
{ appointmentId: { $in: [
  ObjectId("69e963188e990a7c3100da45"),
  ObjectId("69e963188e990a7c3100da46"),
  ObjectId("69e963188e990a7c3100da47"),
  ObjectId("69e963188e990a7c3100da48"),
  ObjectId("69e963188e990a7c3100da49"),
  ObjectId("69e963188e990a7c3100da4a"),
  ObjectId("69e963188e990a7c3100da4b"),
  ObjectId("69e963188e990a7c3100da4c"),
  ObjectId("69e963188e990a7c3100da4d"),
  ObjectId("69e963188e990a7c3100da4e")
] } }

// Conversation items
{ "data.data.object.requestId": { $in: [
  "asyncRequest_2NEAUT0L07000",
  "asyncRequest_2NEAUSVVOD000",
  "asyncRequest_2NEAUT7CO7000",
  "asyncRequest_2NEAUT2QGD000",
  "asyncRequest_2NEAUT5RO7000",
  "asyncRequest_2NEAUT2BG7000",
  "asyncRequest_2NEAUT5AO7000",
  "asyncRequest_2NEAUT778D000",
  "asyncRequest_2NEAUT628D000",
  "asyncRequest_2NEAUT7O87000"
] } }
```

---

## Appendix — Webhook payload used

POST body sent to `/api/webhooks/looker_appointments`. All 10 entries identical except for `appointments.id`. Phone number is the QA Test contact already registered in Spruce (not a real patient).

```json
{
    "data": [
        {
            "appointments.id": "99999951",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        },
        {
            "appointments.id": "99999952",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        },
        {
            "appointments.id": "99999953",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        },
        {
            "appointments.id": "99999954",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        },
        {
            "appointments.id": "99999955",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        },
        {
            "appointments.id": "99999956",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        },
        {
            "appointments.id": "99999957",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        },
        {
            "appointments.id": "99999958",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        },
        {
            "appointments.id": "99999959",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        },
        {
            "appointments.id": "99999960",
            "locations.id": "1229",
            "appointments.status": "Complete",
            "orders.patient_id": "491052",
            "group_providers.npi": "1234567890",
            "appointments.start_local_tz_time": "2026-04-24 10:30:00",
            "orders.created_date_reformat": "2026-04-20",
            "patients.mobile_phone": "9176755175",
            "patients.home_phone": "9176755175",
            "appointments.created_date": "2026-04-22",
            "appointments.order_series_id": "",
            "appointments.first_appt_on_order_series": "No",
            "orders.first_listed_primary_med": ""
        }
    ]
}
```
