## [2026-03-03T15:33:14.858Z] TRIGGER INPUT (bulk route)
```json
{
  "jobName": "recalculate340BByNpi",
  "hospitalSystemId": "69988d1286a333102d620ed8",
  "oldNpis": [
    "4817263950",
    "7390521846",
    "2658401739",
    "9143078265",
    "6025839174"
  ],
  "newNpis": [
    "1710368154",
    "1770865768"
  ],
  "pulseCollection": "pulseJobsAGomez"
}
```

## [2026-03-03T15:33:16.004Z] TRIGGER RESULT (bulk route)
```json
{
  "scheduled": true,
  "affectedNpiCount": 7
}
```

## [2026-03-03T15:33:18.459Z] JOB PROCESSING (worker)
```json
{
  "_id": "69a6ff3ba30cc870dd22a289",
  "name": "recalculate340BByNpi",
  "data": {
    "affectedNpis": [
      "1710368154",
      "1770865768",
      "4817263950",
      "7390521846",
      "2658401739",
      "9143078265",
      "6025839174"
    ],
    "hospitalSystemId": "69988d1286a333102d620ed8"
  },
  "priority": 0,
  "shouldSaveResult": false,
  "attempts": 0,
  "backoff": null,
  "lastModifiedBy": null,
  "lockedAt": "2026-03-03T15:33:18.190Z",
  "type": "normal",
  "nextRunAt": null,
  "lastRunAt": "2026-03-03T15:33:18.400Z"
}
```

## [2026-03-03T15:33:20.492Z] JOB SUCCESS (worker)
```json
{
  "name": "recalculate340BByNpi",
  "_id": "69a6ff3ba30cc870dd22a289",
  "lastFinishedAt": "2026-03-03T15:33:20.437Z"
}
```

