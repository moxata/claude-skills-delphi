---
name: modernize-vcl-edit-to-cx
description: Convert plain VCL text edits and memos in a Delphi form (.dfm + .pas) to DevExpress (cx) editors — TEdit to TcxTextEdit, TDBEdit to TcxDBTextEdit, TMemo to TcxMemo, and TDBMemo to TcxDBMemo — preserving name, position, exact size, tab order, events, dataset and field name. Use when asked to "modernize", "skin", or convert the "TEdit"/"TDBEdit"/"TMemo"/"TDBMemo"/text/memo fields of a VCL form. For an edit that has an image-only browse button beside it, or for buttons, use the sibling skill modernize-vcl-to-devexpress instead.
---

# Modernize plain VCL edits → DevExpress (cx) text editors

Convert the plain text edits of a single Delphi form to their DevExpress (`cx`)
equivalents, matching the established "skin" style of this codebase:

- Unbound `TEdit`  → **`TcxTextEdit`**
- Data-bound `TDBEdit` → **`TcxDBTextEdit`**
- Unbound `TMemo`  → **`TcxMemo`**
- Data-bound `TDBMemo` → **`TcxDBMemo`**

Preserve each control's **name, position (`Left`/`Top`), exact size (`Width`), tab order,
all events, and its dataset + field name**. Font/UI refresh (Tahoma→Segoe UI, sizes,
heights) is **out of scope** — leave fonts and layout alone except where the conversion
forces it (the `Height` change below).

## Scope / detection

**In scope:** a `TEdit`, `TDBEdit`, `TMemo`, or `TDBMemo` that is a normal
text/data field with **no `TSpeedButton` positioned immediately to its right**
(no adjacent browse/ellipsis button). A memo is the multi-line variant — same
treatment as an edit except its `Height` is real layout and is **kept** (see the
`Height` carve-out below).

**Out of scope — defer to the sibling skill `modernize-vcl-to-devexpress`:**
- An edit that has an image-only `TSpeedButton` beside it (`button.Left ≈ edit.Left +
  edit.Width`, same `Top`) → that becomes `TcxButtonEdit`/`TcxDBButtonEdit` there.
- All button conversions (`TBitBtn`/`TSpeedButton`/`TButton` → `TcxButton`).

**Out of scope — leave alone:** `TComboBox`/`TDBComboBox`, `TDBText`, `TLabel`,
`TRadioButton`, `TCheckBox`/`TDBCheckBox`. Do **not** auto-convert obvious date fields:
a `TcxDBDateEdit` exists but is a different mapping — **flag** any date field for the user
rather than turning it into a plain text edit.

## Invariants — always

- Operate on **one form per run**: the matching `<Name>U.dfm` and `<Name>U.pas` pair. Edit
  both files together and keep them consistent — every `.dfm` `object` has a field of the
  same name and type in the form's `type` declaration, and vice versa.
- Make changes on the same rows in the `.dfm` and `.pas` files where possible, to keep the
  diff clean and reviewable.
- **Preserve existing component names and event-handler method names** so wired handlers
  keep working. Do not rename `edt_Naseleno`, `edt_FamiliaKeyPress`, etc.
- When unsure about property style or ordering, **diff against an already-converted form**
  (grep for `TcxDBTextEdit` / `TcxTextEdit`) rather than guessing.

## Transformation — in the `.dfm` (per edit)

1. Change the class: `TEdit` → `TcxTextEdit`, `TDBEdit` → `TcxDBTextEdit`,
   `TMemo` → `TcxMemo`, `TDBMemo` → `TcxDBMemo`.
2. Rewrite data binding (DB edits/memos only):
   - `DataField = 'X'`  → `DataBinding.DataField = 'X'`
   - `DataSource = Y`   → `DataBinding.DataSource = Y`
3. Keep `Left`, `Top`, `TabOrder`, and **every** control-level event (`OnKeyPress`,
   `OnEnter`, `OnExit`, `OnClick`, `OnDblClick`, …) exactly as they were.
   **Exception — `OnChange`:** in cx editors the change event lives on the
   `Properties` object, so move it (see step 5): `OnChange = X` →
   `Properties.OnChange = X`.
4. Emit `Width` as the **last** property of the object (cx style); **drop** `Height`
   (cx-managed) **for single-line edits** (`TcxTextEdit`/`TcxDBTextEdit`). This
   preserves the exact horizontal size while letting the editor manage its own
   height — the only layout change the conversion forces.
   **Memo carve-out:** for memos (`TcxMemo`/`TcxDBMemo`) **keep `Height`** — a memo
   is multi-line and its height is real layout, not cx-managed. Emit both `Height`
   and `Width` (with `Width` still last).
5. Map edit-only properties that don't transfer 1:1 onto `Properties.*`, and **flag each
   one** in the final report so the user can verify it in the IDE:
   - `OnChange = X`           → `Properties.OnChange = X`  (applies to **every**
     conversion — text edits and memos alike; the change event is on `Properties`,
     not the control)
   - `CharCase = ecUpperCase` → `Properties.CharCase = ecUpperCase`
   - `MaxLength = n`          → `Properties.MaxLength = n`
   - `PasswordChar`, `ReadOnly`, etc. → the corresponding `Properties.*`
   Preserve behavior — do **not** silently drop these.
6. Re-emit the converted object at the **END** of its parent container's visual-control
   list — after the other visual controls and before non-visual components such as
   `TFDQuery` / `TDataSource`. This mirrors how the Delphi/DevExpress IDE serializes
   windowed controls (a graphic `TLabel`/`TDBText` is not windowed and stays where it is).

### Canonical before → after (commit `2d3bf94f`)
```
object edt_Naseleno: TDBEdit              object edt_Naseleno: TcxDBTextEdit
  Left = 86                                 Left = 86
  Top = 47                                  Top = 47
  Width = 134                               DataBinding.DataField = 'Naseleno'
  Height = 23                               DataBinding.DataSource = ds_Sobst
  CharCase = ecUpperCase                    TabOrder = 2
  DataField = 'Naseleno'                    Properties.CharCase = ecUpperCase
  DataSource = ds_Sobst                     OnKeyPress = edt_FamiliaKeyPress
  TabOrder = 2                              Width = 134
  OnKeyPress = edt_FamiliaKeyPress        end
end
```
(`Width` last, `Height` gone, binding rewritten to `DataBinding.*`, object moved to the
end of its tab sheet's controls. Note: that commit dropped `CharCase`; per this skill it
is preserved as `Properties.CharCase` and flagged.)

**`OnChange` moves to `Properties` (commit `66ba3fb8`, `SobstEditU.dfm`):**
```
OnChange = edt_BegDateChange   →   Properties.OnChange = edt_BegDateChange
```

### Memo before → after (`TDBMemo` → `TcxDBMemo`)
```
object edt_Adres: TDBMemo                 object edt_Adres: TcxDBMemo
  Left = 16                                 Left = 16
  Top = 304                                 Top = 304
  Width = 407                               DataBinding.DataField = 'Adres'
  Height = 50                               DataBinding.DataSource = ds_Izp
  DataField = 'Adres'                       TabOrder = 15
  DataSource = ds_Izp                       Height = 50
  TabOrder = 15                             Width = 407
end                                       end
```
(Same as an edit **except `Height = 50` is kept** — a memo is multi-line. `Width`
still last; binding rewritten to `DataBinding.*`.)

## Transformation — in the `.pas`

1. In the form's `type` block, change the field's class (`TEdit`→`TcxTextEdit`,
   `TDBEdit`→`TcxDBTextEdit`, `TMemo`→`TcxMemo`, `TDBMemo`→`TcxDBMemo`) and **keep the
   name**. Move the declaration to mirror the `.dfm` reorder — group the converted
   fields with the other already-converted cx edits, just before the `procedure`
   declarations.
2. **No handler changes.** Events are preserved by name (including the one moved to
   `Properties.OnChange`), so do not add, rename, or delete any event-handler
   declaration or body.
3. Ensure the **interface** `uses` clause contains (add any missing, with **no
   duplicates**):
   `cxControls, cxContainer, cxEdit, cxTextEdit, cxMaskEdit`
   and — when any `TDBEdit`/`TDBMemo` was converted — also `cxDBEdit`
   (`TcxDBMemo` is declared there, same as `TcxDBTextEdit`),
   and — when any `TMemo`/`TDBMemo` was converted — also `cxMemo`.
   (Forms already partly skinned with cx button-edits usually have these already; adding
   them is then a no-op.)

## Final self-check & report

Before finishing, verify:

- Every `.dfm` `object` has a matching `.pas` field of the same name **and** type, and
  every field has a matching object.
- None of the converted controls remain `TEdit`/`TDBEdit`/`TMemo`/`TDBMemo`; no
  converted control still carries `DataField` or `DataSource` (must be
  `DataBinding.*`). `Height` is **dropped** on single-line edits but **kept** on memos.
- No converted control still has a control-level `OnChange` — it must be
  `Properties.OnChange`.
- Required units are present in `uses` with no duplicates (`cxMemo` when a memo was
  converted; `cxDBEdit` when any DB control was converted).

Then report a short summary:
- Which controls were converted (old → new type, with name), including memos.
- Every `Properties.*` mapping made (`OnChange`, `CharCase`, `MaxLength`, …) for the
  user to verify in the Delphi IDE; call out memos where `Height` was retained.
- Any field deliberately **skipped** as ambiguous (e.g. a date field that should likely be
  `TcxDBDateEdit`) for the user to handle.

## Reference commits (canonical examples)

- `2d3bf94f` — plain `TDBEdit` (`edt_Naseleno`) → `TcxDBTextEdit`: binding rewrite,
  `Width` last, `Height` dropped, object re-emitted at end of the tab sheet, field
  reordered in the `.pas`.
- `66ba3fb8` — `SobstEditU`: converted `TcxTextEdit`s carry their change event as
  `Properties.OnChange = …` (e.g. `edt_BegDate`, `edt_EndDate`) — the canonical
  `OnChange → Properties.OnChange` move.
- `be1cc14e` — **out-of-scope boundary**: an edit *with* an ellipsis `TSpeedButton`
  (`edt_Oblast`) → `TcxDBButtonEdit`. If you see that pattern, hand off to
  `modernize-vcl-to-devexpress`.
