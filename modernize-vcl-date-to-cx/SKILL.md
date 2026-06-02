---
name: modernize-vcl-date-to-cx
description: Convert date-edit controls in a Delphi form (.dfm + .pas) to DevExpress (cx) date editors — TJvDateEdit / TDateTimePicker / TRzDateTimeEdit and other unbound date pickers to TcxDateEdit, and TJvDBDateEdit / any data-bound date edit to TcxDBDateEdit — preserving name, position, exact width, tab order, events, dataset and field name. As a follow-up it applies the cx date conventions: set Properties.DateOnError = deNull on the cx editors, and rewrite `.Date = 0` / `.Date <> 0` emptiness checks to NullDate (adding unit cxDateUtils). Use when asked to "modernize", "skin", or convert the date / TJvDateEdit / TDateTimePicker fields of a VCL form, or when modernize-vcl-edit-to-cx defers a date field to this skill.
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
   (`TcxDateEdit` and `TcxDBDateEdit` are declared in `cxCalendar`), plus **`cxDateUtils`**
   for the `NullDate` follow-up below. Forms already partly skinned usually have some of
   these — adding is then a no-op. Leaving the old `JvToolEdit` / `Vcl.ComCtrls` units is
   harmless; remove one only if nothing else in the form uses it.
4. **Migrate or flag source-specific code API.** Most code uses `.Date`, which `TcxDateEdit`
   also exposes — those calls keep working. But flag/convert any control-specific member the
   cx editor lacks: `TJvDateEdit.ShowNullDate` / `.Checked`, `TDateTimePicker.DateTime` /
   `.Time`. (Grep the `.pas` for the converted control names to catch these.)

## Secondary — the NullDate convention (once a field is cx)

A cx date editor with `DateOnError = deNull` returns the sentinel **`NullDate`** (from
`cxDateUtils`) when empty/invalid — **not** `0`. So after a control becomes `TcxDateEdit` /
`TcxDBDateEdit` (or for any editor that was already cx), rewrite its emptiness checks:

- `<ctrl>.Date = 0`  → `<ctrl>.Date = NullDate`
- `<ctrl>.Date <> 0` → `<ctrl>.Date <> NullDate`
- reversed order (`0 = <ctrl>.Date`) and the cast form
  `(Control as TcxDateEdit).Date = 0` likewise.

**Only for cx controls.** A still-non-cx date control (`TJvDateEdit`, `TRzDateTimeEdit`,
`TDateTimePicker`) genuinely returns `0` when empty — leave its `= 0` checks alone. So when
converting a *subset* of a form's date fields, change only the comparisons whose control you
actually converted. **Never touch date-vs-date** comparisons (`edt_X.Date <= edt_Y.Date`); in
a mixed expression change only the `= 0` / `<> 0` operand. Inheritance gotcha: a frame may
inherit its `edt_*` fields from a base frame — resolve the type where it is declared
(`TNOIDeathParentFrame` inherits Jv fields from `TNOIBoolFromToDateFrame`).

## How to find candidates

```
grep -nE 'object \w+: (TJvDateEdit|TJvDBDateEdit|TDateTimePicker|TRzDateTimeEdit)' *.dfm
grep -nE 'object \w+: (TcxDateEdit|TcxDBDateEdit)' *.dfm   # already-cx (DateOnError + NullDate only)
grep -nE '\.Date\s*(<>|=)\s*0\b' *.pas                    # emptiness checks for the cx ones
```

## Final self-check & report

- **In-place diff.** `git diff -- <Form>U.dfm <Form>U.pas`: each converted control is a
  **single in-place hunk** (class + properties change where it sits); no `object` shows up
  removed in one place and re-added elsewhere. `.pas` shows only the field-type change, the
  `uses` additions, and `0` → `NullDate` swaps.
- Every `.dfm` `object` has a matching `.pas` field of the same name **and** type, and vice
  versa.
- No converted control remains `TJvDateEdit` / `TJvDBDateEdit` / `TDateTimePicker` /
  `TRzDateTimeEdit`; none still carries bare `DataField`/`DataSource` (must be
  `DataBinding.*`), `Height`, `NumGlyphs`, `ShowNullDate`, `Date`/`Time` literals, or
  `EditType`. No control-level `OnChange` remains (must be `Properties.OnChange`).
- Every converted cx editor has exactly one `Properties.DateOnError = deNull` (not
  duplicated); `Width` is last.
- Re-grep `\.Date\s*(<>|=)\s*0\b`: any remaining hit is a **non-cx** control (not converted
  this run). Required cx units present in `uses` with no duplicates; `cxDateUtils` present
  wherever a `NullDate` comparison was written.
- **Compile** the form in the Delphi IDE: editors stream (no unknown-property error), and
  `NullDate` resolves from `cxDateUtils`.

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
- NullDate follow-up examples (already-cx editors): `EnterETrudZapFormU.pas`,
  `EObezViewerFormU.pas`, `EnterDopSporFormU.pas` (incl. the `(Control as TcxDateEdit)` cast).
