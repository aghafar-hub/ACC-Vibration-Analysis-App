# VIB_ID merge kit

Ready-to-use artifacts for adopting the ACC Platform asset master DB's
`VIB_ID` as a permanent identifier for every RMS/SPM measurement point,
generated from `ACC_PLATFORM_ASSET_MASTER_DB_v3.xlsx`'s `VIB_POINT_MASTER`
tab against the 195 equipment IDs actually present in the live Sheet's RMS
and SPM Registers. Meant for the sandbox Sheet/Web App
(`https://docs.google.com/spreadsheets/d/15wIvMQB_4eYhI9uvx7LZj4Gj4Kpo5vQG7W9U_3_s5Lc/`)
— nothing here touches production.

## The equipment ID format mismatch

The master DB and the live Sheet use different formats for the same
equipment:

| Live Sheet | Master DB |
| ---------- | --------- |
| `111.CP.400` | `111.CP400` |
| `321.MD.140` | `321.MD140` |

i.e. the Sheet has an extra dot before the trailing digits. Stripping it
(`sheetId.replace(/\.(\d+)$/, "$1")`) and looking the result up in the
master DB's `EQUIPMENT_MASTER` tab matched **169 of the Sheet's 195
distinct equipment IDs (86.7%)**.

## Files

- **`vib-point-map-import.csv`** — 1,521 rows, one per (equipment, position,
  RMS-or-SPM point) from `VIB_POINT_MASTER`, filtered to the 169 matched
  equipment and with `Equipment ID` converted back to the Sheet's own dotted
  format so it joins cleanly with the existing RMS/SPM Register and DATA
  rows. Columns: `VIB ID, Equipment ID, Position Code, Family, Point
  Description, Status`. **Import this as a new sheet tab named exactly**
  `🔗 VIB Point Map` (paste starting at cell A1, headers included).
- **`needs-review.csv`** — the 26 equipment IDs that didn't match, with the
  closest same-line-and-type candidates from the master DB where any exist,
  for you to resolve by eye (I did not guess these). Two different kinds of
  gap, worth treating differently:
  - **Numbering drift** (18 of the 26) — e.g. Sheet's `341.BE.040` vs
    master's `341.BE041`, same equipment name ("Bucket elevator"), off by
    one digit. Likely the same physical asset, renumbered when it was
    onboarded into the ACC Platform — worth a quick confirm, not a rebuild.
  - **Not yet onboarded** (8 of the 26, including `212.HC.100` and the four
    `645.BL.*` Blowers) — no candidate at all in the master DB for that
    line+type combination. These need to be added to the ACC Platform
    asset master first; nothing here can invent VIB_IDs for them.
- **`../Code.v2.gs`** — `Code.fixed.gs` (the already bug-corrected backend)
  plus reading this new tab into `readAll()`'s response as `vibPoints`. A
  3-line diff — see the file's own header comment. Paste into the sandbox
  Apps Script project (**not** production), then Deploy → Manage
  deployments → New version → Deploy.

## What's NOT done yet

This kit gets the data into the sandbox Sheet and makes it readable by the
backend. It does **not** yet:

- Change anything in `src/` — the React app doesn't request or use
  `vibPoints` yet.
- Attach a `VIB_ID` to any existing RMS DATA/SPM DATA history row.
- Change how New Reading's point picker works.

Those are the next steps, once you've confirmed the sandbox tab imports
cleanly and the 26 review rows are resolved (or explicitly deferred). The
plan from there: New Reading's point picker switches from free text to a
dropdown sourced from `vibPoints` (filtered to the selected equipment),
carrying `vibId` onto every new reading going forward — existing history
gets backfilled separately, matched by equipment + point description, with
anything ambiguous flagged rather than guessed.
