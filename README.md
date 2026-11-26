# f13ld.beam
Custom Implicit Beam Lattices

🔗 **[Launch → F13LD.beam](https://mshomper.github.io/f13ld.beam)**

<img width="873" height="1263" alt="beamscreenshot" src="https://github.com/user-attachments/assets/e61f3397-cb05-46d9-9f6b-32de1561dd54" />

## Using it

Open `index.html` in a browser with WebGL. One file, no build step.

The controls sit in four sections, one at a time: **Cell**, **Draw**, **Geometry** and **Analyze**. On desktop a rail beside the panel switches them. On a phone the panel is a bottom sheet with a tab bar; Draw lowers the sheet so the snap points stay tappable. Every (i) opens its help card, and the card's **show** button jumps to the control.

1. **Cell** — choose **Library** or **Builder** (there are no tabs). Library: pick a topology and tiling. Builder: start from an empty cell and draw your own. Cell size sits next to the export.
2. **Draw** — Builder only. Place beams between snap points, set snap density and symmetry, Undo / Redo / Clear.
3. **Geometry** — beam radius and tiling.
4. **Analyze** — homogenization: relative density and the directional stiffness surfaces. E_s and ν_s set the base material. Stiffness is solved shortly after you stop dragging; results dim until it finishes.
5. **Recipe** menu in the header: Open recipe…, Export JSON, Export LTCX, Export CSV, Queue, Choose queue…, Mesh, Lab. It follows the active mode: the recipe is tagged `beam` on Library and `beam-builder` on Builder, and Mesh / Lab / Queue send that one.

The ghost preview (the beam you are about to place, with its symmetry copies) is always on.

## Importing a recipe

**Recipe → Open recipe…**, dropping a `.json` file on the page, or pasting recipe JSON loads it back. The header then says *Imported … · Undo*; Undo restores the previous design and mode.

- Works with either mode's recipe; the tool switches to match.
- E_s and ν_s are restored. Homogenization results are not; they are recomputed.
- A recipe from another F13LD tool is refused with that tool's name.
- Values outside the allowed range are clamped, with a warning.
- Unknown keys are ignored.

## Rendering

- Quality is automatic; there are no quality buttons.
- While anything moves, the view renders at an adaptive reduced scale and is upscaled. About 150 ms after it stops, one full-resolution rest frame is drawn in strips.
- Pixel ratio goes up to 2.
- The render loop sleeps when idle.
- A lost WebGL context is rebuilt without a reload.

## Controls

The panel uses [liquid-ui](https://github.com/mshomper/liquid-ui) widgets (sliders, dropdowns, segmented controls, toggles, Recipe menu), drawn in WebGL over the native inputs, which stay the source of state. Without WebGL the native controls are shown instead.

## Export

Both exports carry `meta.version` (`0.5.0`) and `meta.tool` (`beam` or `beam-builder`).
