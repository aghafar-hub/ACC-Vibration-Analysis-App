# Google Sheet schema

Everything in this app lives in one Google Sheet. This document lays out
every tab this app reads or writes, and exactly which column maps to which
field in `src/parsers.js`. Two confidence levels appear below, and each
section says which applies:

- **Confirmed** — the client sends or receives this literal column header
  text, so the mapping is exact (found directly in `index-XLkmhznS.js`).
- **Inferred** — the client never sees this tab's raw columns (a dedicated
  backend action hides the sheet layout entirely, e.g. `updateCompliance`,
  `appendAction`). The _shape of the data exchanged_ is confirmed from the
  bundle; the underlying sheet's exact tab name and column layout is a
  reasonable inference, not a verified fact — the Apps Script source itself
  isn't part of this repo.

## 📥 RMS DATA (confirmed)

One row per RMS (vibration velocity) reading. Sheet tab name is literally
`"📥 RMS DATA"` — the emoji is part of the real tab name, sent as the
`sheet` parameter on every append/updateRow/deleteRow call for this data.

| Col | Header                | Field (`rowToRMS`) | Notes                                                                                                               |
| --- | --------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------- |
| 0   | `#`                   | `seq`              | Row sequence number, assigned client-side as `max(existing)+1`                                                      |
| 1   | `Equipment Name`      | `equipmentName`    |                                                                                                                     |
| 2   | `Equipment ID`        | `equipmentId`      | Match column for update/delete                                                                                      |
| 3   | `Asset ID`            | `point`            | The measurement point label (e.g. "Motor DE"), normalized on read — see `normalizePoint()`                          |
| 4   | `Date`                | `date`             | Match column for update/delete                                                                                      |
| 5   | `AXial (mm/s)`        | `axial`            | Note the header's own capitalization: "AXial", not "Axial"                                                          |
| 6   | `Gear\n(mm/s)`        | `gear`             | The header cell has a literal line break in it (wrapped text) — the lookup key must match exactly, newline included |
| 7   | `Horizontal (mm/s)`   | `horizontal`       |                                                                                                                     |
| 8   | `Vertical (mm/s)`     | `vertical`         |                                                                                                                     |
| 9   | `Max Velocity (mm/s)` | `maxVel`           | If blank on read, computed client-side as `max(axial, gear, horizontal, vertical)`                                  |

Match key for `updateRow`/`deleteRow`: columns `[2, 3, 4]` (Equipment ID,
Asset ID, Date).

## 📥 SPM DATA (confirmed)

One row per SPM (Shock Pulse Method bearing-condition) reading. Sheet tab
name is literally `"📥 SPM DATA"`.

| Col | Header           | Field (`rowToSPM`) | Notes                           |
| --- | ---------------- | ------------------ | ------------------------------- |
| 0   | `#`              | `seq`              |                                 |
| 1   | `Equipment Name` | `equipmentName`    |                                 |
| 2   | `Equipment ID`   | `equipmentId`      | Match column                    |
| 3   | `Asset ID`       | `point`            | Measurement point, normalized   |
| 4   | `Type`           | `type`             | Defaults to `"SPM"` if blank    |
| 5   | `Date`           | `date`             | Match column                    |
| 6   | `HDm (dBsv)`     | `hdm`              | Maximum shock value             |
| 7   | `HDc (dBsv)`     | `hdc`              | Carpet (background) shock value |
| 8   | `Gs`             | `gs`               |                                 |

Match key: columns `[2, 3, 5]` (Equipment ID, Asset ID, Date).

## Equipment Register (inferred: two sides, one page)

The Equipment Register page joins two separately-synced datasets by
`Equipment ID` — `readAll()`'s `rmsRegister` and `spmRegister` arrays. The
client never references either tab's name directly (`updateRegisterLimits`
takes a `type: "RMS" | "SPM"` instead of a `sheet` name, so the backend owns
that mapping) — whether this is one sheet with all columns or two separate
tabs isn't confirmed from the client bundle alone. Column names below are
confirmed (they're read by exact header text); the tab split is inferred.

**RMS side** (`rowToRmsRegister`):

| Header           | Field           | Default if blank                                                 |
| ---------------- | --------------- | ---------------------------------------------------------------- |
| `Equipment ID`   | `equipmentId`   |                                                                  |
| `Equipment Name` | `equipment`     |                                                                  |
| `Name Plate`     | `namePlate`     |                                                                  |
| `Eq Type`        | `eqType`        |                                                                  |
| `Line`           | `line`          |                                                                  |
| `RMS Good`       | `rmsGood`       | `2.8`                                                            |
| `RMS Acceptable` | `rmsAcceptable` | `7.1`                                                            |
| `RMS Alarm`      | `rmsAlarm`      | `18`                                                             |
| `Points`         | `points`        | Comma-separated point list, e.g. `"Motor DE, Motor NDE, Fan DE"` |

**SPM side** (`rowToSpmRegister`):

| Header           | Field         | Default if blank |
| ---------------- | ------------- | ---------------- |
| `Equipment ID`   | `equipmentId` |                  |
| `Equipment Name` | `equipment`   |                  |
| `Line`           | `line`        |                  |
| `Points`         | `points`      | Comma-separated  |
| `SPM Type`       | `spmType`     |                  |
| `SPM Normal`     | `spmNormal`   | `20`             |
| `SPM Caution`    | `spmCaution`  | `35`             |
| `SPM Alarm`      | `spmAlarm`    | `50`             |

`updateRegisterLimits` (fired from both Equipment Register's Edit modal and
Limits Settings) sends `points` back as the **raw comma-separated string**
from the form, not the parsed/normalized array — whatever whitespace or
casing the user typed is what lands in the sheet.

## Last RMS Reading / Last SPM Reading (inferred)

Two more sheets, one row per equipment+point, holding just the _most
recent_ reading — this is what the Dashboard reads instead of scanning the
full RMS DATA/SPM DATA history on every load. Written via dedicated
`upsertLastRMS`/`upsertLastSPM`/`deleteLastRMS`/`deleteLastSPM` actions
(again no `sheet` name — the backend owns it) and populated in bulk by
Settings → System → "Backfill Last Readings" (`backfillLastReadings`,
which the UI describes as scanning all RMS & SPM DATA history and writing
the latest-per-equipment-per-point row to these sheets).

**Last RMS** (`rowToLastRMS`) fields: `equipmentId`, `equipment`, `line`,
`point`, `date`, `axial`, `gear`, `horizontal`, `vertical`, `maxVel`
(read from `Max Velocity` or `Max Velocity (mm/s)` — either header works),
`readingStatus` (the computed Good/Acceptable/Alarm/Danger band, stamped by
the client at write time), `machineStatus` (a separate field the backend
appears to own — populated by something other than a plain reading save;
never set to a non-empty value by this app's own write path, always sent as
`""` on `upsertLastRMS`/`upsertLastSPM`). Match key: `[0, 3]` (Equipment ID,
Point).

**Last SPM** (`rowToLastSPM`): same shape, with `spmType`, `hdm`, `hdc`,
`gs` instead of the RMS fields.

`machineStatus` is worth calling out: the Compliance Tracker page and
Action Tracker both read it (as the "current machine status" driving the
compliance tiles and the Generate Monthly Actions candidate list), but
`upsertLastRMS`/`upsertLastSPM`'s own payload doesn't include a
`machineStatus` field at all — only `readingStatus` (the client-computed
Good/Acceptable/Alarm/Danger band) is sent. The client's own in-memory copy
of a freshly-saved reading sets `machineStatus: ""` locally, but that value
is never transmitted. Either the backend derives/stamps `machineStatus`
itself on write (e.g. from the register's thresholds, server-side), or it's
populated by some other process entirely outside this app. Not confirmed
either way — flagging it rather than guessing.

## Compliance Tracker (inferred)

`readAll()`'s `compliance` array already arrives pre-shaped as one object
per equipment: `{ equipmentId, line, equipment, last, months: [{ month:
"YYYY-MM", status }] }`. The client does no column parsing for this one —
`rowToCompliance()` just defends the shape. The most likely underlying
layout (by analogy with the sibling oil-analysis app's own Sample Tracker
sheet, and because a wide one-row-per-equipment-many-month-columns layout
is the natural way to hand-maintain this in a spreadsheet) is one row per
equipment with a column per month, each cell holding a free-text status —
but this is a guess about the sheet shape, not something the client
confirms. `last` is whatever the sheet's own "most recent status" cell (or
formula) holds; a blank/missing `last` is treated by `classifyComplianceStatus()`
as `"Alert"`, not `"Missing"` — see `domain.js`.

Each month's `status` cell is free text, not a fixed enum —
`classifyComplianceStatus()` (`domain.js`) buckets it: `yes`/`normal`/`good`
→ Normal, `no` → Alert, `missing` → Missing, `alarm`/`danger` → themselves,
anything containing "caution"/"observation"/"under"/"comment" → Caution,
anything else → Other.

## Action Tracker (confirmed field names, inferred tab name)

`appendAction`/`updateAction` send a named-field object (not a row array +
headers, unlike RMS/SPM DATA) — so every field name below is confirmed
directly from the client. Columns, in the order the CSV export
(`ACTION_HEADERS` in `parsers.js`) uses:

| Field               | Notes                                                        |
| ------------------- | ------------------------------------------------------------ |
| `Action No`         | Format `V-###` (e.g. `V-014`), zero-padded to 3 digits       |
| `Equipment ID`      |                                                              |
| `Equipment Name`    |                                                              |
| `Line`              |                                                              |
| `Reading Date`      |                                                              |
| `Trigger Type`      | `"RMS"`, `"SPM"`, or `"Both"`                                |
| `Trigger Point`     | Free text — the measurement point that triggered this action |
| `Trigger Value`     | Free text — the reading value, as a string                   |
| `Machine Status`    | `"Good"` / `"Acceptable"` / `"Alarm"` / `"Danger"`           |
| `Revision Date`     |                                                              |
| `Action Status`     | `"Open"` / `"In Progress"` / `"Closed"`                      |
| `Completion Date`   |                                                              |
| `Contractor`        | One of the comma-separated contractor list from Settings     |
| `Contractor Action` | Free text                                                    |
| `ACC Action`        | Free text                                                    |
| `Agreed Action`     | Free text                                                    |

**Action numbering**: `readLastActionNo` is the backend's own source of
truth (`{ status: "ok", nextNo, maxNum }`); if it can't be reached, the
client falls back to `max(existing V-### numbers) + 1`.

**`updateAction`'s payload is inconsistent with `appendAction`'s** — see
[API_CONTRACT.md](./API_CONTRACT.md#known-gaps) for the reproduced-as-found
detail.

## Configuration (confirmed — documented in-app)

Settings → Configuration itself describes this sheet's exact shape (not
just inferred from wire traffic): a tab named exactly `Configuration`, with
header row `Key | Value`. The app auto-populates rows on first
`saveConfig` call; there's no fixed key list documented beyond what
`saveConfig`'s payload sends: `webhookUrl`, `googleSheetUrl`, `contractors`.
