---
name: modernize-vcl-edit-to-cx
description: Convert plain VCL text edits, memos, and combo boxes in a Delphi form (.dfm + .pas) to DevExpress (cx) editors — TEdit to TcxTextEdit, TDBEdit to TcxDBTextEdit, TMemo to TcxMemo, TDBMemo to TcxDBMemo, TComboBox to TcxComboBox, TDBComboBox to TcxDBComboBox — preserving name, position, exact size, tab order, events, dataset and field name. Use when asked to "modernize", "skin", or convert the "TEdit"/"TDBEdit"/"TMemo"/"TDBMemo"/text/memo fields of a VCL form. For an edit that has an image-only browse button beside it, or for buttons, use the sibling skill modernize-vcl-to-devexpress instead.
---

# Modernize plain VCL edits → DevExpress (cx) text editors

Convert the plain text edits, memos, and combo boxes of a single Delphi form to their
DevExpress (`cx`) equivalents, matching the established "skin" style of this codebase:

- Unbound `TEdit`  → **`TcxTextEdit`**
- Data-bound `TDBEdit` → **`TcxDBTextEdit`**
- Unbound `TMemo`  → **`TcxMemo`**
- Data-bound `TDBMemo` → **`TcxDBMemo`**
- Unbound `TComboBox` → **`TcxComboBox`**
- Data-bound `TDBComboBox` → **`TcxDBComboBox`**

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

**Out of scope — leave alone:** `TDBText`, `TLabel`,
`TRadioButton`, `TCheckBox`/`TDBCheckBox`. Do **not** auto-convert obvious date fields:
a `TcxDBDateEdit` exists but is a different mapping — **flag** any date field for the user
rather than turning it into a plain text edit.

## Invariants — always

- Operate on **one form per run**: the matching `<Name>U.dfm` and `<Name>U.pas` pair. Edit
  both files together and keep them consistent — every `.dfm` `object` has a field of the
  same name and type in the form's `type` declaration, and vice versa.
- **Convert each control in place — same rows, no relocation.** Overwrite the control's
  existing `object … end` block *where it already sits*; never move, reorder, regroup, or
  re-emit it elsewhere. Mechanically this is a **single replacement of that one block** —
  not a delete here plus an add somewhere else. Change only the class name, the inner
  properties, and the data binding. Inner property lines may shift within the object, but
  the object/field block stays where it was, so the diff reads as a clean in-place
  replacement on the original rows. The same applies in the `.pas`: change only the class
  token on the existing field line; never move the field.
- **Preserve existing component names and event-handler method names** so wired handlers
  keep working. Do not rename `edt_Naseleno`, `edt_FamiliaKeyPress`, etc.
- When unsure about property style or property ordering, **diff against an already-converted
  form** (grep for `TcxDBTextEdit` / `TcxTextEdit`) rather than guessing — but use it as a
  reference for **property style, within-object property ordering, and `Properties.*`
  mappings only, never for where the object lives**. Those forms were produced the legacy
  way, with converted controls re-emitted at the **end** of the list; you keep yours in
  place on the same rows.

## Transformation — in the `.dfm` (per edit)

**Edit mechanics — in place.** Convert each edit by a *single replacement of its existing
`object … end` block* — match the block where it already sits and overwrite it there. Do
**not** delete the edit and emit a new `object` for it anywhere else (the end of the
container's control list, just before the non-visual components, etc.). The objects
immediately above and below the converted block stay byte-for-byte unchanged. The numbered
steps below describe what the in-place replacement block contains.

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
6. **Leave the object where it is — do not move it within its parent container's component
   list.** Edit it in place so the converted control stays on the same rows as the original.

### Canonical before → after — shown as an in-place diff

The conversion is a same-rows replacement of the object's block. Below is the verified
`edt_Razdane` conversion in `SobstEditU.dfm`; the neighbour `cb_InVedom` directly after it
is **untouched** — the block changes class and properties *where it sits*:

```
       end
-      object edt_Razdane: TDBEdit
+      object edt_Razdane: TcxDBTextEdit
         Left = 129
         Top = 97
-        Width = 73
-        Height = 23
-        DataField = 'LN4_DataRazdane'
-        DataSource = ds_Sobst
+        DataBinding.DataField = 'LN4_DataRazdane'
+        DataBinding.DataSource = ds_Sobst
         TabOrder = 6
+        Width = 73
       end
       object cb_InVedom: TCheckBox        <- unchanged, still immediately after
```

A correct conversion **never** shows a removed `object … end` in one place and an added one
elsewhere. Here `Width` became the last property, `Height` was dropped (single-line edit),
and the binding moved to `DataBinding.*`.

Properties that don't transfer 1:1 land on `Properties.*` **inside the same block**, e.g.
`CharCase = ecUpperCase` → `Properties.CharCase = ecUpperCase`, and
`OnChange = edt_BegDateChange` → `Properties.OnChange = edt_BegDateChange`
(commit `66ba3fb8`, `SobstEditU.dfm`).

### Memo before → after — in-place diff (`TDBMemo` → `TcxDBMemo`)

Same as an edit **except `Height` is kept** — a memo is multi-line and its height is real
layout. Verified `edt_Adres` conversion in `SobstEditU.dfm`, again in place:

```
-      object edt_Adres: TDBMemo
+      object edt_Adres: TcxDBMemo
         Left = 86
         Top = 74
-        Width = 354
-        Height = 41
-        DataField = 'Adres'
-        DataSource = ds_Sobst
+        DataBinding.DataField = 'Adres'
+        DataBinding.DataSource = ds_Sobst
         TabOrder = 4
         OnKeyPress = edt_AdresKeyPress
+        Height = 41
+        Width = 354
       end
```
(`Height` **and** `Width` kept, `Width` last; binding rewritten to `DataBinding.*`.)

## Transformation — combo boxes (`TComboBox` / `TDBComboBox`)

Combo boxes follow the same in-place rule as edits, with these differences:

1. Change the class: `TComboBox` → `TcxComboBox`, `TDBComboBox` → `TcxDBComboBox`.
2. **Drop** `Style = csDropDownList` and `Height = 23`.
   Replace with `Properties.DropDownListStyle = lsFixedList`.
3. Move `OnChange = X` → `Properties.OnChange = X` (same rule as for edits).
4. Move `Items.Strings = (...)` → `Properties.Items.Strings = (...)`.
   Drop the `Text = ...` line (it duplicates the `ItemIndex` selection and is not
   used by `TcxComboBox`).
5. **`ItemIndex` — keep in DFM, but place it after `Properties.Items.Strings`.**
   The Delphi streaming system applies properties in DFM order. If `ItemIndex` appears
   before the items are loaded it is silently discarded and the IDE drops it on next
   save. Always emit `ItemIndex` **after** `Properties.Items.Strings`.
   - Keep `ItemIndex` if the original value is `≥ 0` (i.e., something was selected).
   - Drop `ItemIndex` only if the original value was `-1` (nothing selected — that is
     the `TcxComboBox` default and does not need to be written).
6. Rewrite data binding for DB combos (same as DB edits):
   `DataField = 'X'` → `DataBinding.DataField = 'X'`,
   `DataSource = Y`  → `DataBinding.DataSource = Y`.
7. Emit `Width` as the **last** property. Drop `Height`.

**Canonical property order inside the converted block:**

```
object cb_Foo: TcxComboBox
  Left = …
  Top = …
  Properties.DropDownListStyle = lsFixedList
  Properties.Items.Strings = (           ← only if originally had Items.Strings
    '…'
    '…')
  ItemIndex = …                          ← AFTER Items.Strings; omit only if was -1
  Properties.OnChange = cb_FooChange     ← only if originally had OnChange
  TabOrder = …
  Width = …
end
```

**In the `.pas` — code references:**
Anywhere the code calls `.Items.Count` or `.Items.Add(…)` / `.Items.Clear` on a
converted combo, update to `.Properties.Items.Count`, `.Properties.Items.Add(…)`,
`.Properties.Items.Clear`. Properties accessed directly (`ItemIndex`, `Visible`,
`Enabled`, `SetFocus`, `Clear` for text) stay as-is.

### Combo box before → after — in-place diff

Verified `cb_FlagCertificate` conversion in `NewEObezFormU.dfm`
(commits `f74cab1d`, `ca406be4`):

```
-  object cb_FlagCertificate: TComboBox
+  object cb_FlagCertificate: TcxComboBox
     Left = 24
     Top = 177
-    Width = 409
-    Height = 23
-    Style = csDropDownList
     ItemIndex = 0
-    TabOrder = 3
-    Text = '1 - …'
-    OnChange = cb_FlagCertificateChange
-    Items.Strings = (
-      '1 - …'
-      '2 - …'
-      '3 - …')
+    Properties.DropDownListStyle = lsFixedList
+    Properties.Items.Strings = (
+      '1 - …'
+      '2 - …'
+      '3 - …')
+    Properties.OnChange = cb_FlagCertificateChange
+    TabOrder = 3
+    Width = 409
   end
```

## Transformation — in the `.pas`

1. In the form's `type` block, change the field's class (`TEdit`→`TcxTextEdit`,
   `TDBEdit`→`TcxDBTextEdit`, `TMemo`→`TcxMemo`, `TDBMemo`→`TcxDBMemo`) and **keep the
   name**. Leave the declaration in its original position — do not move or regroup it, so
   the change stays on the same row.
2. **No handler changes.** Events are preserved by name (including the one moved to
   `Properties.OnChange`), so do not add, rename, or delete any event-handler
   declaration or body.
3. Ensure the **interface** `uses` clause contains (add any missing, with **no
   duplicates**):
   `cxControls, cxContainer, cxEdit, cxTextEdit, cxMaskEdit`
   and — when any `TDBEdit`/`TDBMemo` was converted — also `cxDBEdit`
   (`TcxDBMemo` is declared there, same as `TcxDBTextEdit`),
   and — when any `TMemo`/`TDBMemo` was converted — also `cxMemo`,
   and — when any `TComboBox` was converted — also `cxDropDownEdit`
   (`TcxComboBox` is declared there),
   and — when any `TDBComboBox` was converted — also `cxDBEdit`
   (`TcxDBComboBox` is declared there, same as `TcxDBTextEdit`).
   (Forms already partly skinned with cx controls usually have some of these already;
   adding them is then a no-op.)

## Final self-check & report

Before finishing, verify:

- **In-place diff (do this first).** Run `git diff -- <Form>U.dfm <Form>U.pas`. Every
  converted control must appear as a **single in-place hunk** — the `object`/field line
  changes class while the objects around it stay byte-for-byte unchanged. If a control shows
  up as a removed `object … end` block in one hunk and an added block in another (moved to
  the end of the list, or anywhere else), it was **relocated**: undo the move and redo it as
  an in-place block replacement. Same-row, in-place editing is a hard requirement, not
  "where possible."
- Every `.dfm` `object` has a matching `.pas` field of the same name **and** type, and
  every field has a matching object.
- None of the converted controls remain `TEdit`/`TDBEdit`/`TMemo`/`TDBMemo`/
  `TComboBox`/`TDBComboBox`; no converted control still carries bare `DataField` or
  `DataSource` (must be `DataBinding.*`). `Height` is **dropped** on single-line edits
  and combo boxes, but **kept** on memos.
- No converted control still has a control-level `OnChange` — it must be
  `Properties.OnChange`.
- Combo boxes: no `Style = csDropDownList`, no bare `Items.Strings` (must be
  `Properties.Items.Strings`), no bare `Text = …` line. `ItemIndex` is present and
  placed **after** `Properties.Items.Strings` (omitted only if original was `-1`).
  Code references `.Items.*` updated to `.Properties.Items.*`.
- Required units are present in `uses` with no duplicates (`cxMemo` when a memo was
  converted; `cxDBEdit` when any DB control was converted; `cxDropDownEdit` when any
  `TComboBox` was converted).

Then report a short summary:
- Which controls were converted (old → new type, with name), including memos.
- Every `Properties.*` mapping made (`OnChange`, `CharCase`, `MaxLength`, …) for the
  user to verify in the Delphi IDE; call out memos where `Height` was retained.
- Any field deliberately **skipped** as ambiguous (e.g. a date field that should likely be
  `TcxDBDateEdit`) for the user to handle.

## Reference commits (canonical examples)

Diff these for **property-level style only** — they predate the in-place rule and
re-emitted converted controls at the end of the list, so ignore what they show for object
positioning; convert in place on the same rows regardless.

- `2d3bf94f` — plain `TDBEdit` (`edt_Naseleno`) → `TcxDBTextEdit`: binding rewrite,
  `Width` last, `Height` dropped (the commit also re-emitted the object at the end of the
  tab sheet and reordered the `.pas` field — **do not** copy that; convert in place).
- `66ba3fb8` — `SobstEditU`: converted `TcxTextEdit`s carry their change event as
  `Properties.OnChange = …` (e.g. `edt_BegDate`, `edt_EndDate`) — the canonical
  `OnChange → Properties.OnChange` move.
- `be1cc14e` — **out-of-scope boundary**: an edit *with* an ellipsis `TSpeedButton`
  (`edt_Oblast`) → `TcxDBButtonEdit`. If you see that pattern, hand off to
  `modernize-vcl-to-devexpress`.
- `f74cab1d` — `NewEObezFormU`: `cb_VidUdost: TComboBox` → `TcxComboBox`; adds
  `cxDropDownEdit` to `uses`; `Style` removed, `Properties.DropDownListStyle = lsFixedList`,
  `OnChange` moved to `Properties.OnChange`.
- `ca406be4` — `NewEObezFormU` fix: `.Items.Count` → `.Properties.Items.Count`,
  `.Items.Add(…)` → `.Properties.Items.Add(…)` in code after the combo conversion above.
