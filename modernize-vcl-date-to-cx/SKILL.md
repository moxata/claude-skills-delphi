---
name: modernize-vcl-date-to-cx
description: Convert date-edit controls in a Delphi form (.dfm + .pas) to DevExpress (cx) date editors — TJvDateEdit / TDateTimePicker / TRzDateTimeEdit and other unbound date pickers to TcxDateEdit, and TJvDBDateEdit / any data-bound date edit to TcxDBDateEdit — preserving name, position, exact width, tab order, events, dataset and field name. As a follow-up it applies the cx date conventions: set Properties.DateOnError = deNull on the cx editors, and rewrite every `.Date` usage (reads, writes, and `= 0` / `<> 0` emptiness checks) on a cx editor to the `.DateForStorage` property from ZapiRoutines (which maps empty↔0), never referencing NullDate. Use when asked to "modernize", "skin", or convert the date / TJvDateEdit / TDateTimePicker fields of a VCL form, or when modernize-vcl-edit-to-cx defers a date field to this skill.
---

# Modernize VCL date edits → DevExpress (cx) date editors

Convert the date-picker controls of a single Delphi form to their DevExpress (`cx`)
equivalents, matching the established "skin" style of this codebase. This is the date-field
counterpart to `modernize-vcl-edit-to-cx` (which converts text/memo/combo and **defers** date
fields here).

| Source control | → Target | Bound? |
|---|---|---|
| `TJvDateEdit` (JvToolEdit) | **`TcxDateEdit`** | unbound |
| `TDateTimePicker` (Vcl.ComCtrls) | **`TcxDateEdit`** | unbound |
| `TRzDateTimeEdit` (Raize/Konopka) | **`TcxDateEdit`** | unbound |
| `TJvDBDateEdit` | **`TcxDBDateEdit`** | data-bound |
| any other data-bound `T*DBDateEdit` | **`TcxDBDateEdit`** | data-bound |

**Rule of thumb:** a control that carries `DataField` + `DataSource` (or whose type ends in
`DBDateEdit`) → `TcxDBDateEdit`; everything else → `TcxDateEdit`.

Preserve each control's **name, position (`Left`/`Top`), exact width (`Width`), tab order, all
events, and its dataset + field name**. Font/UI refresh is out of scope — leave fonts/layout
alone except the `Height` drop the conversion forces.

## Invariants — always

- Operate on **one form per run**: the matching `<Name>U.dfm` and `<Name>U.pas` pair. Keep
  them consistent — every `.dfm` `object` has a field of the same name and type in the form's
  `type` declaration, and vice versa.
- **Convert each control in place — same rows, no relocation.** Overwrite the control's
  existing `object … end` block *where it already sits* (a single block replacement); never
  move, reorder, or re-emit it elsewhere. Same for the `.pas`: change only the class token on
  the existing field line; never move the field.
- **Preserve component names and event-handler method names** so wired handlers keep working.
  Do not rename `edt_OtData`, `edt_OtDataChange`, etc.
- When unsure about cx property style/ordering, **diff against an already-converted date
  editor** in this repo (`grep -nE 'object \w+: (TcxDateEdit|TcxDBDateEdit)' *.dfm`) — e.g.
  `EnterETrudZapFormU.dfm` (DB) or `EObezViewerFormU.dfm` (unbound) — for property style only,
  not for object position (keep yours in place).

## Transformation — in the `.dfm` (per date edit, in place)

1. **Change the class** per the table (bound → `TcxDBDateEdit`, unbound → `TcxDateEdit`).
2. **Rewrite data binding** (bound only):
   `DataField = 'X'` → `DataBinding.DataField = 'X'`,
   `DataSource = Y` → `DataBinding.DataSource = Y`.
3. **Keep** `Left`, `Top`, `TabOrder`, and every control-level event (`OnEnter`, `OnExit`,
   `OnClick`, …). **Exception — `OnChange`:** in cx editors the change event lives on
   `Properties`, so move it: `OnChange = X` → `Properties.OnChange = X`.
4. **Drop `Height`** (cx-managed) and emit **`Width` last** (cx style).
5. **Drop source-specific properties that don't transfer**, and **flag each drop** in the
   report:
   - `TJvDateEdit` / `TJvDBDateEdit`: drop `NumGlyphs`, `ShowNullDate`, `Glyph`,
     `ButtonWidth`, `DateFormat`, `Flat`. (`ShowNullDate = False` is effectively replaced by
     the `DateOnError = deNull` convention added in step 7.)
   - `TDateTimePicker`: drop the design-time `Date = …` / `Time = …` literals, `Kind`,
     `Format`, `DateMode`. **Flag for the user** if `Kind = dtkTime` — that is a *time*
     picker; `TcxDateEdit` is date-only, so it likely needs `TcxTimeEdit` (or
     `Properties.Kind = ckDateTime`) instead — do not silently convert a time picker to a
     date editor.
   - `TRzDateTimeEdit`: drop `EditType = etDate`. **Flag** if `EditType` is `etDateTime` /
     `etTime` (same date-only caveat).
6. **Leave the object where it is** — edit in place; neighbour objects stay byte-for-byte
   unchanged.
7. **Apply the cx date convention:** add `Properties.DateOnError = deNull` inside the block
   (group it with any other `Properties.*` lines, after `DataBinding.*`, before `TabOrder` /
   events / `Width`). If a `Properties.DateOnError` already exists, set its value to `deNull`
   rather than duplicating.

### Canonical before → after (in-place diffs)

`TJvDateEdit` → `TcxDateEdit` (`edt_OtData`, `eObezFrameU.dfm`):
```
-  object edt_OtData: TJvDateEdit
+  object edt_OtData: TcxDateEdit
     Left = 3
     Top = 68
-    Width = 121
-    Height = 21
-    NumGlyphs = 2
-    ShowNullDate = False
+    Properties.DateOnError = deNull
+    Properties.OnChange = edt_OtDataChange
     TabOrder = 1
-    OnChange = edt_OtDataChange
+    Width = 121
   end
```

`TJvDBDateEdit` → `TcxDBDateEdit` (`edt_lkdata`, `EditIzpulnitelFormU.dfm`):
```
-  object edt_lkdata: TJvDBDateEdit
+  object edt_lkdata: TcxDBDateEdit
     Left = 152
     Top = 200
-    Width = 121
-    Height = 25
-    DataField = 'Li4naKData'
-    DataSource = ds_Izp
-    NumGlyphs = 2
-    ShowNullDate = False
+    DataBinding.DataField = 'Li4naKData'
+    DataBinding.DataSource = ds_Izp
+    Properties.DateOnError = deNull
     TabOrder = 10
+    Width = 121
   end
```

`TDateTimePicker` → `TcxDateEdit` (`edt_OtData`, `DnevBolnFormU.dfm`) — drop `Date`/`Time`:
```
-  object edt_OtData: TDateTimePicker
+  object edt_OtData: TcxDateEdit
     Left = 48
     Top = 128
-    Width = 105
-    Height = 25
-    Date = 41421.000000000000000000
-    Time = 0.457718275501974900
+    Properties.DateOnError = deNull
     TabOrder = 4
+    Width = 105
   end
```

`TRzDateTimeEdit` → `TcxDateEdit` (`edt_DateNew`, `UvedCh123BojoFormU.dfm`):
```
-  object edt_DateNew: TRzDateTimeEdit
+  object edt_DateNew: TcxDateEdit
     Left = 224
     Top = 133
-    Width = 145
-    Height = 24
-    EditType = etDate
+    Properties.DateOnError = deNull
     TabOrder = 4
+    Width = 145
   end
```

## Transformation — in the `.pas`

1. In the form's `type` block, change the field's class to `TcxDateEdit` / `TcxDBDateEdit`
   and **keep the name**, in its original position (no move/regroup).
2. **No handler changes.** Events are preserved by name (including the one moved to
   `Properties.OnChange`); do not add, rename, or delete any handler declaration or body.
3. Ensure the **interface** `uses` clause contains (add any missing, **no duplicates**):
   `cxControls, cxContainer, cxEdit, cxCalendar, cxDropDownEdit`
   (`TcxDateEdit` and `TcxDBDateEdit` are declared in `cxCalendar`), plus **`ZapiRoutines`**
   (home of the `DateForStorage` helpers used in the follow-up below — usually added to the
   *implementation* `uses`, matching the codebase). `cxDateUtils` is **no longer needed for
   code** — keep it only if already present for another reason. Forms already partly skinned
   usually have some of these — adding is then a no-op. Leaving the old `JvToolEdit` /
   `Vcl.ComCtrls` units is harmless; remove one only if nothing else in the form uses it.
4. **Migrate `.Date`, or flag source-specific API.** Migrate **every** `.Date` read / write /
   compare on a converted (or already-cx) editor to **`.DateForStorage`** — `.Date` must never
   remain on a cx editor (see the convention section below). But flag/convert any
   control-specific member the cx editor lacks: `TJvDateEdit.ShowNullDate` / `.Checked`,
   `TDateTimePicker.DateTime` / `.Time`. (Grep the `.pas` for the converted control names to
   catch both.)

## Secondary — the DateForStorage convention (once a field is cx)

A cx date editor with `DateOnError = deNull` represents **empty** with the internal sentinel
`NullDate` (-700000), while the DB / NOI layer uses **`0`**. To bridge the two, this codebase
gives the cx editors a single `DateForStorage: TDateTime` property (the `TcxDateEditHelper` /
`TcxDBDateEditHelper` class helpers in **`ZapiRoutines`**) that speaks the storage convention
(`0` = empty) and hides `NullDate`: reading maps `NullDate`→`0`, writing maps `0`→`NullDate`.
So once a control is cx (or for any editor that was already cx), **route all date access
through `DateForStorage`; `.Date` and the `NullDate` constant must never appear in code.**

- emptiness checks: `<ctrl>.Date = 0` → `<ctrl>.DateForStorage = 0`; `<ctrl>.Date <> 0` →
  `<ctrl>.DateForStorage <> 0` (reversed order and the cast form
  `(Control as TcxDateEdit).Date = 0` → `(Control as TcxDateEdit).DateForStorage = 0` likewise).
- reads: `v := <ctrl>.Date` → `v := <ctrl>.DateForStorage`; `f(<ctrl>.Date)` →
  `f(<ctrl>.DateForStorage)`; field reads `<ctrl>.Date := ds.FieldByName('X').AsDateTime` →
  `<ctrl>.DateForStorage := ds.FieldByName('X').AsDateTime` (the setter does the 0→empty
  conversion, so a `FieldDate(...)`-style wrapper is no longer needed — use plain
  `.AsDateTime`).
- writes / clears: `ds.FieldByName('X').AsDateTime := <ctrl>.Date` →
  `:= <ctrl>.DateForStorage`; `<ctrl>.Date := 0` and `<ctrl>.Date := NullDate` →
  `<ctrl>.DateForStorage := 0`.
- date-vs-date comparisons among cx controls **are migrated too** (this revises the old
  "never touch date-vs-date" rule): `edt_X.Date <= edt_Y.Date` →
  `edt_X.DateForStorage <= edt_Y.DateForStorage`.
- **var cascade:** when a local/record `TDateTime` is read from an editor and later compared to
  the empty sentinel, that comparison must follow the storage convention too — after
  `v := <ctrl>.DateForStorage`, a downstream `if v = NullDate` / `if v <> NullDate` becomes
  `if v = 0` / `if v <> 0`.

**Only for cx controls.** A still-non-cx date control (`TJvDateEdit`, `TRzDateTimeEdit`,
`TDateTimePicker`) genuinely returns `0` when empty and has **no** `DateForStorage` helper —
leave its `.Date` and `= 0` checks **exactly as they are**. So when converting a *subset* of a
form's date fields, migrate only the access whose control you actually converted; in a mixed
expression change only the cx operand. **Never write the `NullDate` constant or `.Date =
NullDate` in code.** Inheritance gotcha: a frame may inherit its `edt_*` fields from a base
frame — resolve the type where it is declared (`TNOIDeathParentFrame` inherits Jv fields from
`TNOIBoolFromToDateFrame`, so those stay `.Date = 0`).

## How to find candidates

```
grep -nE 'object \w+: (TJvDateEdit|TJvDBDateEdit|TDateTimePicker|TRzDateTimeEdit)' *.dfm
grep -nE 'object \w+: (TcxDateEdit|TcxDBDateEdit)' *.dfm   # already-cx (DateOnError + DateForStorage only)
grep -nE '\.Date\s*(<>|=)\s*0\b' *.pas                    # cx-editor emptiness checks → rewrite to .DateForStorage
grep -nE '<edt>\.Date\b' *.pas                            # per converted control: every .Date access to migrate
grep -nE 'NullDate' *.pas                                 # guard: should be none on a migrated cx editor
```

## Final self-check & report

- **In-place diff.** `git diff -- <Form>U.dfm <Form>U.pas`: each converted control is a
  **single in-place hunk** (class + properties change where it sits); no `object` shows up
  removed in one place and re-added elsewhere. `.pas` shows only the field-type change, the
  `uses` additions, and `.Date` → `.DateForStorage` swaps.
- Every `.dfm` `object` has a matching `.pas` field of the same name **and** type, and vice
  versa.
- No converted control remains `TJvDateEdit` / `TJvDBDateEdit` / `TDateTimePicker` /
  `TRzDateTimeEdit`; none still carries bare `DataField`/`DataSource` (must be
  `DataBinding.*`), `Height`, `NumGlyphs`, `ShowNullDate`, `Date`/`Time` literals, or
  `EditType`. No control-level `OnChange` remains (must be `Properties.OnChange`).
- Every converted cx editor has exactly one `Properties.DateOnError = deNull` (not
  duplicated); `Width` is last.
- Re-grep `<edt>\.Date\b` for each converted control: **no cx-editor `.Date` remains** (all
  migrated to `.DateForStorage`). Re-grep `NullDate`: none on a migrated cx editor (only
  unrelated internals — XML/parser returns, string literals — may remain). Any remaining
  `\.Date\s*(<>|=)\s*0\b` hit must be a **non-cx** control (not converted this run). Required
  cx units present in `uses` with no duplicates, and **`ZapiRoutines`** present (home of
  `DateForStorage`).
- **Compile** the form in the Delphi IDE: editors stream (no unknown-property error), and
  `DateForStorage` resolves from `ZapiRoutines`.

Then report: each control converted (old → new type, with name); every dropped property and
every `Properties.*` move (`OnChange`, `DateOnError`) for the user to verify in the IDE; any
field **flagged** rather than converted (a `dtkTime` / `etDateTime` time picker); and any
`.Date = 0` check left as `0` because its control stayed non-cx.

## Reference points (live in this codebase)

- Conversion sources: `TJvDateEdit` in `eObezFrameU` and the `NOI*` frames; `TJvDBDateEdit` in
  `EditIzpulnitelFormU` (`edt_lkdata`, `edt_Razdane`), `DeklCh73FormU` (`edt_BirthDate`),
  `UMain` (`edt_SuperBegDate`); `TDateTimePicker` in `DnevBolnFormU`; `TRzDateTimeEdit` in
  `UvedCh123BojoFormU` (`edt_DateNew`).
- Target shape to copy property style from: `EnterETrudZapFormU.dfm` / `EObezViewerFormU.dfm`
  (cx date editors; the latter has one with an existing `Properties.OnChange`).
- `DateForStorage` follow-up examples (already-cx editors): `EnterETrudZapFormU.pas`,
  `EObezViewerFormU.pas`, `EnterDopSporFormU.pas` (incl. the
  `(Control as TcxDateEdit).DateForStorage = 0` cast), `Ch222FormU.pas` /
  `NewPlatOtpFormU.pas` (the `begDate`/`endDate` var cascade). The helpers themselves live in
  `ZapiRoutines.pas` (`TcxDateEditHelper` / `TcxDBDateEditHelper`).
