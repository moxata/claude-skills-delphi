---
name: modernize-vcl-to-devexpress
description: Convert plain VCL controls in a Delphi form (.dfm + .pas) to DevExpress cx editors/buttons — TBitBtn/TSpeedButton to TcxButton, and an edit field with an image-only SpeedButton beside it to TcxButtonEdit/TcxDBButtonEdit. Use when asked to "modernize", "skin", or "bitbtn"-convert a VCL form.
---

# Modernize VCL form → DevExpress (cx) editors & buttons

Convert the plain-VCL controls of a single Delphi form to their DevExpress (`cx`)
equivalents, matching the established "skin" style of this codebase. Two transformations
are in scope:

- **A.** `TBitBtn` and standalone command `TSpeedButton` → `TcxButton`.
- **B.** A `TDBEdit`/`TEdit` with an image-only `TSpeedButton` immediately to its right
  → `TcxDBButtonEdit` / `TcxButtonEdit`.

Font/UI refresh (Tahoma→Segoe UI, sizes, control heights) is **out of scope** — leave
fonts and layout alone except where a conversion forces a change.

## Invariants — always

- Operate on **one form per run**: the matching `<Name>U.dfm` and `<Name>U.pas` pair.
  Both files must be edited together and kept consistent — every `.dfm` `object` has a
  field of the same name and type in the form's `type` declaration, and vice versa.
- **Convert each control in place — same rows, no relocation.** Overwrite the control's
  existing `object … end` block *where it already sits*; never move, reorder, regroup, or
  re-emit it elsewhere. Mechanically this is a **single replacement of that one block** —
  not a delete here plus an add somewhere else (e.g. at the end of the visual-control list).
  Change only the class name, the inner properties, and the data binding. Inner property
  lines may shift within the object, but the object/field block stays where it was, so the
  diff reads as a clean in-place replacement on the original rows. The same applies in the
  `.pas`: change only the class token on the existing field line; never move the field.
  (The one object that legitimately disappears is a Transformation-B browse `TSpeedButton`,
  which is absorbed into its edit — see B.)
- **Preserve existing component names and event-handler method names** so wired handlers
  keep working. Do not normalize `btn_Save`→`btn_Ok` etc.
- The image list reference is always `ZapiImagesDataModule.img_btn16`. Any unit that gains
  a `TcxButton` therefore needs `ZapiImagesDataModuleU` in its **implementation** `uses`.
- When unsure about an image index, an edit/button mapping, or property style, **diff
  against an already-converted form** rather than guessing. Many forms are already done
  (grep for `OptionsImage.Images = ZapiImagesDataModule.img_btn16`). Copy their property
  style and image indices only — those forms re-emitted converted controls at the **end**
  of the list under the old convention; you do **not** relocate anything, so convert in
  place on the same rows.

## Transformation A — TBitBtn / standalone TSpeedButton → TcxButton

Applies to `TBitBtn`, and to any `TSpeedButton` used as a normal command button (it has a
real `Caption` and is **not** sitting immediately to the right of an edit — that case is
Transformation B).

### In the `.dfm`

**In place:** overwrite each button's existing `object … end` block where it sits — a single
block replacement, never a delete-here-plus-add-elsewhere. The objects above and below stay
unchanged.

For each such object:

1. Change the class to `TcxButton`.
2. Delete `Glyph.Data`, `NumGlyphs`, `Kind`, `Flat`.
3. Add:
   ```
   OptionsImage.ImageIndex = <n>
   OptionsImage.Images = ZapiImagesDataModule.img_btn16
   ```
4. Primary / affirmative button: add `Default = True`.
5. Close / cancel button: add `Cancel = True` and `ModalResult = 2`. If its `OnClick`
   body did nothing but set `ModalResult := mrCancel`, **remove that handler** and rely on
   the `Cancel`/`ModalResult` properties instead.
6. Standalone non-default action buttons (Insert, Delete, …): add `TabStop = False`.
7. **Leave the object where it is — do not move it within the form's component list.**
   Edit it in place so the converted button stays on the same rows as the original.
   A trailing numeric suffix on a name (`btn_Remove1`) is acceptable only if a name clash
   actually forces it; otherwise keep the original name.

**In-place diff** (real `btn_Insert` in `SobstEditU.dfm`; the `btn_Delete` neighbour right
after it is untouched — the block changes *where it sits*):

```
       end
-      object btn_Insert: TSpeedButton
+      object btn_Insert: TcxButton
         Left = 8
         Top = 10
         Width = 89
         Height = 36
         Caption = #1053#1086#1074'...'
-        Flat = True
-        Glyph.Data = {... binary ...}
+        OptionsImage.ImageIndex = 5
+        OptionsImage.Images = ZapiImagesDataModule.img_btn16
+        TabOrder = 1
+        TabStop = False
         OnClick = btn_InsertClick
       end
       object btn_Delete: TcxButton        <- neighbour object untouched, same position
```

A correct conversion **never** shows a removed `object … end` in one place and an added one
elsewhere.

Sizes seen in the codebase for dialog OK/Cancel buttons are roughly `Width = 98;
Height = 36`, but keep the original geometry unless asked to restyle.

#### Image-index convention
Verify against a converted form or `ZapiImagesDataModuleU.dfm` when unsure. Observed:

| Index | Meaning | Typical caption |
|------:|---------|-----------------|
| 0  | primary / OK / Save (pair with `Default = True`) | Запис, Добре, Избор, Създаване, Обработка |
| 1  | Cancel / Close (pair with `Cancel = True`)        | Отказ, Изход |
| 5  | New / Insert | Нов… |
| 6  | Delete       | Изтриване |
| 21 | Previous     | Предишен |
| 22 | Next         | Следващ |

### In the `.pas`
1. In the form's `type` block, change the field's class from `TBitBtn`/`TSpeedButton` to
   `TcxButton` (keep the name; leave the declaration in its original position — do not
   reorder it).
2. Add to the **interface** `uses`:
   `cxGraphics, cxLookAndFeels, cxLookAndFeelPainters, cxButtons`.
3. Add to the **implementation** `uses`: `ZapiImagesDataModuleU`.
4. Delete any handler you removed in the `.dfm` step (both its forward declaration in the
   `type` block and its implementation body).

## Transformation B — edit + adjacent image-only SpeedButton → Tcx(DB)ButtonEdit

**Detect the pattern:** a `TDBEdit` or `TEdit` that has a `TSpeedButton` positioned
immediately to its right (`button.Left ≈ edit.Left + edit.Width`, same `Top`), where the
SpeedButton carries a glyph and **no meaningful caption** — i.e. a browse/ellipsis button
for that field. A SpeedButton with its own real caption is *not* this pattern (treat it
under Transformation A).

- Data-bound `TDBEdit` → **`TcxDBButtonEdit`**.
- Unbound `TEdit` → **`TcxButtonEdit`**.

### In the `.dfm`

**In place:** convert the edit by overwriting its existing `object … end` block where it
sits. The one object that legitimately disappears is the **adjacent browse `TSpeedButton`**,
which is absorbed into the edit's `Properties.Buttons` — delete that button object, but do
**not** move the edit.

1. Change the edit's class to the cx type.
2. Rewrite data binding:
   - `DataField = 'X'`  → `DataBinding.DataField = 'X'`
   - `DataSource = Y`   → `DataBinding.DataSource = Y`
3. Replace the separate SpeedButton with an embedded button on the edit, and delete the
   SpeedButton object:
   ```
   Properties.Buttons = <
     item
       Default = True
       Kind = bkEllipsis
     end>
   Properties.OnButtonClick = <editname>PropertiesButtonClick
   ```
   (`<editname>` = the edit's component name, e.g. `edt_Oblast` → `edt_OblastPropertiesButtonClick`.)
4. Keep `OnEnter` / `OnExit` / `OnKeyPress` and similar events.
5. Emit `Width` as the **last** property of the object (cx style); drop the now-managed
   `Height`.
6. `CharCase` is not carried over directly — if the field needs forced uppercase, use
   `Properties.CharCase = ecUpperCase` and **flag this to the user** rather than silently
   dropping it.

**In-place diff** (illustrative `edt_Oblast`; the edit stays put, only the adjacent browse
button block is removed):

```
       end
-      object edt_Oblast: TDBEdit
+      object edt_Oblast: TcxDBButtonEdit
         Left = 86
         Top = 47
-        Width = 134
-        Height = 23
-        DataField = 'Oblast'
-        DataSource = ds_Sobst
+        DataBinding.DataField = 'Oblast'
+        DataBinding.DataSource = ds_Sobst
+        Properties.Buttons = <
+          item
+            Default = True
+            Kind = bkEllipsis
+          end>
+        Properties.OnButtonClick = edt_OblastPropertiesButtonClick
         TabOrder = 2
+        Width = 134
       end
-      object btn_Oblast: TSpeedButton        <- adjacent browse button: removed (absorbed)
-        Left = 220
-        Top = 47
-        Glyph.Data = {... binary ...}
-        OnClick = btn_OblastClick
-      end
       object <next control>: …               <- every other object untouched
```

### In the `.pas`
1. In the `type` block: remove the `TSpeedButton` field; change the edit's class to the cx
   type (keep the edit's name; leave its declaration where it is).
2. Convert the SpeedButton's old `OnClick` handler into the button-click handler:
   ```pascal
   procedure T<Form>.<editname>PropertiesButtonClick(Sender: TObject; AButtonIndex: Integer);
   ```
   Update both the forward declaration in the `type` block and the implementation
   signature. Remove the old SpeedButton handler declaration.
3. **Rewrite every data-source reference on that edit** throughout the unit:
   `<edit>.DataSource` → `<edit>.DataBinding.DataSource`. Grep the whole unit for
   `<edit>.DataSource` so none are missed (they appear in the button handler and often in
   OnEnter/OnExit logic).
4. Add to the **interface** `uses`:
   `cxControls, cxContainer, cxEdit, cxTextEdit, cxMaskEdit, cxButtonEdit` and — for
   DB edits — `cxDBEdit`.

## Final self-check & report

Before finishing, verify:

- **In-place diff (do this first).** Run `git diff -- <Form>U.dfm <Form>U.pas`. Every
  converted control must appear as a **single in-place hunk** — the `object`/field line
  changes class while the objects around it stay byte-for-byte unchanged. If a control shows
  up as a removed `object … end` block in one hunk and an added block in another (moved to
  the end of the list, or anywhere else), it was **relocated**: undo the move and redo it as
  an in-place block replacement. The only object that should fully disappear is a
  Transformation-B browse `TSpeedButton`. Same-row, in-place editing is a hard requirement.
- Every `.dfm` `object` has a matching `.pas` field of the same name **and** type, and
  every field has a matching object.
- No `TBitBtn` remains; no `Glyph.Data`/`NumGlyphs`/`Kind` lingers on a converted control;
  no orphaned `TSpeedButton` field or dangling/duplicate event handler.
- Required units were added to `uses`, with **no duplicate** unit names.

Then report a short summary:
- Which components were converted (old → new type, with name).
- Any handlers removed or renamed.
- Any `ImageIndex` values you guessed, and any properties dropped (e.g. `CharCase`,
  custom `Height`) that the user should verify in the Delphi IDE.

## Reference commits (canonical examples)

Diff these to confirm style on edge cases. **Caveat:** these commits predate the in-place
rule and re-emitted converted controls at the end of the component list — **ignore that
repositioning**; copy only their property-level style and convert in place on the same rows.

- `9695a553` — minimal `TButton`/`TBitBtn` → `TcxButton`.
- `451429c2` — `TBitBtn` Save/Close pair → `TcxButton` (Default + Cancel/ModalResult,
  cancel-only handler deleted).
- `4c2fe5ba` — standalone `TSpeedButton` Insert/Delete → `TcxButton` (`TabStop = False`,
  ImageIndex 5/6).
- `be1cc14e` — `TDBEdit` + ellipsis `TSpeedButton` (`edt_Oblast`/`btn_Oblast`) →
  `TcxDBButtonEdit`, incl. handler rename to `PropertiesButtonClick` and all
  `.DataSource` → `.DataBinding.DataSource` rewrites.
- `ae3dcdb5` — `TBitBtn` Default/Cancel pair → `TcxButton`.
