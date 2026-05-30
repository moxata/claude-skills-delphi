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
- Make changes on the same rows in the `.dfm` and `.pas` files where possible, to keep 
  the diff clean and reviewable.
- **Preserve existing component names and event-handler method names** so wired handlers
  keep working. Do not normalize `btn_Save`→`btn_Ok` etc.
- The image list reference is always `ZapiImagesDataModule.img_btn16`. Any unit that gains
  a `TcxButton` therefore needs `ZapiImagesDataModuleU` in its **implementation** `uses`.
- When unsure about an image index, an edit/button mapping, or property style, **diff
  against an already-converted form** rather than guessing. Many forms are already done
  (grep for `OptionsImage.Images = ZapiImagesDataModule.img_btn16`).

## Transformation A — TBitBtn / standalone TSpeedButton → TcxButton

Applies to `TBitBtn`, and to any `TSpeedButton` used as a normal command button (it has a
real `Caption` and is **not** sitting immediately to the right of an edit — that case is
Transformation B).

### In the `.dfm`
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
7. **Re-emit the converted object at the END of the form's top-level visual-component
   list** — after the other visual controls and before non-visual components such as
   `TFDQuery` / `TDataSource`. (This mirrors how the Delphi/DevExpress IDE writes them.)
   A trailing numeric suffix on a name (`btn_Remove1`) is acceptable only if a name clash
   actually forces it; otherwise keep the original name.

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
   `TcxButton` (keep the name; reorder to mirror the `.dfm` if you reordered there).
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

### In the `.pas`
1. In the `type` block: remove the `TSpeedButton` field; change the edit's class to the cx
   type (keep the edit's name).
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

Diff these to confirm style on edge cases:

- `9695a553` — minimal `TButton`/`TBitBtn` → `TcxButton`.
- `451429c2` — `TBitBtn` Save/Close pair → `TcxButton` (Default + Cancel/ModalResult,
  cancel-only handler deleted).
- `4c2fe5ba` — standalone `TSpeedButton` Insert/Delete → `TcxButton` (`TabStop = False`,
  ImageIndex 5/6).
- `be1cc14e` — `TDBEdit` + ellipsis `TSpeedButton` (`edt_Oblast`/`btn_Oblast`) →
  `TcxDBButtonEdit`, incl. handler rename to `PropertiesButtonClick` and all
  `.DataSource` → `.DataBinding.DataSource` rewrites.
- `ae3dcdb5` — `TBitBtn` Default/Cancel pair → `TcxButton`.
