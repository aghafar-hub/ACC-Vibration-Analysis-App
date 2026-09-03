# VIB_ID merge kit

Rebuilds the vibration app's equipment/point data around the ACC Platform
asset master DB (`ACC_PLATFORM_ASSET_MASTER_DB_v3.xlsx`) as the single
source of truth — **not** a reconciliation between two ID schemes. Per your
direction: the master DB's equipment ID format (`111.CP400`) is the real
one going forward; the old Sheet's format (`111.CP.400`) is retired.
Targets the sandbox
(`https://docs.google.com/spreadsheets/d/15wIvMQB_4eYhI9uvx7LZj4Gj4Kpo5vQG7W9U_3_s5Lc/`)
— nothing here touches production.

## Scope and decisions (as you specified)

- **Equipment covered**: only equipment with vibration points actually
  defined in `VIB_POINT_MASTER` — **165** equipment (after normalizing a
  handful of inconsistently-formatted IDs within the master file itself,
  e.g. `645.FN952` and `645.FN.952` both appearing for the same asset).
- **Thresholds**: standard defaults for everyone — RMS Good/Acceptable/Alarm
  `2.8 / 7.1 / 18` mm/s, SPM Normal/Caution/Alarm `20 / 35 / 50` dBsv (what
  ISO 10816 and your own sheet already use almost universally). These are
  written as real values into the rebuilt Register, not left as an
  app-side fallback — same as today, still editable per equipment from
  Equipment Register / Limits Settings afterward.
- **Historical data**: migrated in place — existing RMS/SPM DATA history
  keeps its equipment's trend continuity under the new ID.

## Coverage

Of the 165 target equipment: **158 matched** an equipment already in your
current Sheet (by comparing both IDs' canonical form), so their existing
name/type/line carries over and their history migrates. **7 are newly
covered** — vibration points exist for them in the master DB but they
weren't in your Sheet before — sourced from the master DB's generic
description, flagged for a better name. Nothing was left unresolved or
guessed.

## Files, in the order to apply them

1. **`equipment-id-migration-map.csv`** — reference only (158 old→new ID
   pairs). This is the same mapping baked into `Migrate.gs` below; the CSV
   is here so you can eyeball it before running anything.
2. **`Migrate.gs`** — paste as an _additional_ file in the sandbox's Apps
   Script project (alongside `Code.v2.gs` — Apps Script projects hold
   multiple `.gs` files). **Run these 5 functions one at a time**, as 5
   separate Run clicks (pick each from the editor's function dropdown), not
   the combined `migrateEquipmentIds()` — on a sheet this size, one big run
   burns most of its execution time budget on the first large sheet and
   every sheet after it dies with "Service timed out: Spreadsheets" before
   it can even start:
   - `migrateRmsData()`
   - `migrateSpmData()`
   - `migrateLastRmsReading()`
   - `migrateLastSpmReading()`
   - `migrateComplianceTracker()`

   Check View → Executions (or View → Logs right after each run) for that
   sheet's change count — it also logs progress every 1,000 rows so a run
   in progress isn't a silent wait. Renames Equipment ID cells in
   `📥 RMS DATA`, `📥 SPM DATA`, `📋 Last RMS Reading`, `📋 Last SPM
   Reading`, and `📋 Compliance Tracker` (column C) from old IDs to new.
   Safe to re-run any of them — anything already on a new-format ID (or
   already renamed by an earlier partial run) is left alone, so if one
   fails partway through, just run that same function again. Does **not**
   touch the Register tabs (step 3 replaces those wholesale instead).
3. **`rms-register-rebuild.csv`** / **`spm-register-rebuild.csv`** — the
   full replacement content for `⚙ RMS Register` / `⚙ SPM Register`
   (165 and 158 rows — 7 of the 165 have RMS points but no SPM points
   defined in the master DB, hence the count difference). Select all
   existing content in each tab and paste these over it entirely (headers
   included).
4. **`vib-point-map-import.csv`** (1,765 rows) — paste as a new tab named
   exactly `🔗 VIB Point Map`, same as before, except equipment IDs are now
   in native master format directly (no conversion needed anymore).
5. **`new-equipment-needs-naming.csv`** — the 7 newly-covered equipment,
   each with the master DB's generic description as a placeholder name
   (e.g. `465.BL580` → "Blower / rotary blower"). Worth a quick pass to
   give them proper names in the rebuilt Register before relying on them —
   not blocking, just cosmetic.
6. **`Code.v2.gs`** (in the parent `apps-script/` folder, not this one) —
   the backend addition that reads the new VIB Point Map tab into
   `readAll()`. Paste into the sandbox project, Deploy → Manage
   deployments → New version → Deploy.

## Post-migration audit

**`remaining-old-format-audit.csv`** — after running the migration and
pasting the rebuilt Register/VIB Point Map, I re-read the sandbox sheet and
regex-scanned it for every remaining old-format ID (`\d{3}\.[A-Za-z]{2,3}\.\d+`),
cross-checked each against the migration map and master DB, and classified
all 199 distinct ones found:

- **155 `EDIT`** — already in `equipment-id-migration-map.csv`, just
  weren't renamed yet in whichever sheet they're in. This is the bulk of
  it — re-run the 4 remaining `Migrate.gs` functions (`migrateSpmData`,
  `migrateLastRmsReading`, `migrateLastSpmReading`,
  `migrateComplianceTracker`) and check each one's log actually reports a
  nonzero rename count, not a `FAILED` line.
- **21 `ADD (needs vib points)`** — real equipment in `EQUIPMENT_MASTER`
  (e.g. all of `761/762.WI.21x` — 15 Water injection units) with zero
  vibration points defined in `VIB_POINT_MASTER`. Needs someone to add
  their points to the master DB, or tell me the points directly.
- **17 `ADD (not in master DB)`** — not in the master DB at all, e.g. the
  `441/442.WI.00x` series and `212.HC.100`'s close cousin `645.BL.58x`.
  Needs full onboarding (name, line, points) before anything can be built
  for these.
- **4 `ADD (mirror twin)`** — e.g. `322.MD.152` (30 cells of real history)
  has no master DB points, but its Line1 twin `321.MD152` has an identical
  point structure already defined. Very likely just a master DB gap for
  that specific line, not a different asset — but I didn't invent points
  for these without you confirming it's actually the same equipment type.
- **2 `ADD (has vib points, wasn't in old sheet)`** — `533.MD.301` /
  `534.MD.301` already have master DB points, just weren't part of your
  original Register (note: NOT the same as `533.MD.302`/`534.MD.302`,
  which already are — worth double-checking these aren't a typo of each
  other in your historical data).

## What's still not done

No `src/` changes yet — the React app doesn't request or use `vibPoints`,
and New Reading's point picker is still free text. Once you've run the
migration and rebuild above and confirmed the sandbox looks right, the next
step is wiring the app: New Reading's point picker sourced from
`vibPoints` (carrying `vibId` onto every new reading), Equipment
Register/Graphs showing VIB_ID per point, and only then switching
`DEFAULT_WEBHOOK_URL` over from production to this sandbox once you're
satisfied — that's a decision for you to make explicitly, not something
this kit does on its own.
