# F13LD.beam UI audit (stage 1)

Branch `ui-refresh`, 2026-11-25. Read-only: no code changed. No `CLAUDE.md` in this repo, so the brief applies unchanged.
Screenshots: `docs/ui-audit/` (Playwright, local Chrome, swiftshader).

| Screenshot | Shows |
|---|---|
| `desktop-library.png` | 1440×900, Library tab, top of panel |
| `desktop-library-panel-bottom.png` | Library, stiffness readouts and export row |
| `desktop-library-full.png` | full page |
| `desktop-builder.png` / `desktop-builder-panel-bottom.png` | Builder tab (Kelvin loaded via "Edit this cell") |
| `desktop-view-menu.png` | view menu open |
| `desktop-help.png` | help drawer |
| `phone-library.png` / `phone-library-panel-bottom.png` / `phone-library-full.png` | 390×844 Library |
| `phone-builder.png` | 390×844 Builder: overlapping rows |

## 1. Current state

- **Version:** header `v0.4.0` (line 147). Export `meta.version` is `'0.2.1'` on both paths (lines 2416, 3247).
- **Structure:** single `index.html`, 3893 lines.

| Block | Lines | What |
|---|---|---|
| `<style>` | 7–118 | app CSS, hex colours, no tokens |
| markup | 119–443 | header, tab bar (Library / Builder), viewer `#cw`, `#panel` with `#libraryPanel` and `#builderPanel` |
| main `<script>` | 444–3472 | F13LD-SHADE (445–491), F13LD-VIEW drop-in (493–562), catalog (22 topologies, 568), homogenization (DSM-PBC), `StiffnessSurface` (1187–1408), `Raymarcher` (1588–2136), clip/gizmo (2138–2297), handlers, exports, `Builder` (2815–3179), symmetry ops, init (3426–3471) |
| help drop-in | 3474–3847 | `style#f13h-theme`, `style#f13h-style`, `script#f13h-script` (v1.2) |
| queue client | 3848–3891 | loads `f13ld-queue.js` from `mshomper.github.io`, mounts Queue split button before `.fh-mesh-btn`; `alert()` fallback (3879) |

- **Drop-ins present:** help (v1.2), queue client, view menu (F13LD-VIEW v1), shared shading (F13LD-SHADE v1). No liquid-ui, no `F13UI`, no recipe import.
- **Renderer:** WebGL2 ray-march of a capsule union. Beam endpoints are uniform arrays `beamA/beamB[256]`, `beamCount`, `uRadius`, `uTile`, clip uniforms. One fragment shader; step count baked from the quality tier. Two more WebGL2 contexts for the stiffness surfaces (`#stiffSurfCv`, `#bldStiffSurfCv`).
- **Two modes, two export paths. Never rename either `meta.tool`:**

`beam` (Library, `buildExportJSON()`, line 2411):
```
{ meta:{version:'0.2.1', tool:'beam', timestamp},
  topology:{name, beam_count},
  geometry:{radius, cell},
  metrics:{relative_density_pct, total_beam_length_local},
  beams:[[ax,ay,az,bx,by,bz], …],          // cell-local [-1,1]
  homogenization:{method, voigt_convention, valid, material:{Es_MPa, nu_s},
                  counts:{…}, stiffness_C_MPa, compliance_S_MPa_inv,
                  engineering_constants_MPa, poisson_ratios, anisotropy} | null }
```

`beam-builder` (Builder, `bldBuildExportData()`, line 3238):
```
{ meta:{version:'0.2.1', tool:'beam-builder', timestamp},
  cell:{name, beam_count},
  geometry:{radius, cell:1.5},              // cell is fixed at 1.5
  metrics:{…}, beams:[…], homogenization:{…} }
```

- `bldOpenInMesh()` (3292) builds a third, shorter `beam-builder` payload (no metrics/homogenization) but nothing calls it.
- Header Mesh / Lab (`openInMesh`, `openInLab`, 2728–2740) and the queue (`getRecipe: buildExportJSON`, 3870) **always send the Library recipe**, even on the Builder tab.
- Queue item tag is `tool:'f13ld.beam'` (3870). This is the queue's own field, separate from `meta.tool`.
- Not exported: tiling, quality, clip, snap density, symmetry, edit mode, ghost.
- Existing import path: `?b=<beam-builder JSON>` URL param (3446–3467) loads beams, name and radius into the Builder.

## 2. Findings

| # | Issue | Where | Severity | Rule |
|---|---|---|---|---|
| F1 | Phone Builder rows overlap: Add/Remove sits over snap buttons, radius label over Remove, tiling over Clear, "+ New" clipped | 321–382 (inline flex rows, `min-width:0`) | High | no horizontal scroll; 44 px; one focus |
| F2 | Control column scrolls sideways at 390: panel `scrollWidth` exceeds width by 18 px (Library) and 29 px (Builder). `.viz-row3` is 96+160+96 px plus gaps; Builder rows can't shrink | 87, 321–382 | High | no horizontal scroll, ever |
| F3 | All controls in one long scrolling panel per tab; viewer squeezed to 318 px tall at 1440×900 (`#panel max-height:52vh`), 255 px on phone | 10, 21 | High | one section at a time (lesson 6); one focus |
| F4 | Header Mesh / Lab / Queue send the Library recipe while the user is on Builder | 145–146, 2728–2740, 3870 | High (functional) | recipe contract |
| F5 | Render loop never sleeps: rAF runs forever; both stiffness surfaces redraw every frame (hidden tab included), DPR uncapped. Measured 32 draws/s at idle with auto-rotate off | 2108–2131, 1359–1384, 1403 | High | lesson 2; render-quality §4 |
| F6 | Quality switch Low/Med/High/Ultra recompiles the shader (2 compiles per change); single tier, no adaptive scale, no strip rest frame, `preserveDrawingBuffer:true`, default antialias | 186–189, 1637–1641, 2336–2355, 1593 | Medium | lesson 2; render-quality §2–3 |
| F7 | Slider ticks run full DSM-PBC homogenization + stiffness surface update synchronously. Median 0.6–0.8 ms, max 22 ms (Library, F2BCC-Z) and 42 ms (Builder, octahedral) in swiftshader | 2304–2313, 3223–3227 | Medium | respond within 400 ms; lesson 1 spirit |
| F8 | `cell size` slider changes nothing on screen or in metrics (shader and density are cell-local); it only feeds export, LTCX and CSV. Still triggers homogenization per tick | 238, 2304, 2418, 2755 | Low | fewer choices; tidy output |
| F9 | Text below 11 px: 22 elements at 9 px, 22 at 10 px, 7 at 7.5 px (sub/sup labels) | 24, 47, 49, 63, 96, 104, 109, 117, 247, 253, 302, 431; help CSS 3515, 3547, 3563, 3578 | Medium | type scale 11/13/15/20/28 |
| F10 | Targets under 44 px: 44 controls measured (header pills 25 px, tabs 31 px, clip pills 21 px, quality 19 px, mode buttons 20 px, help (i) 14 px, slider tracks 5 px, checkbox 13 px) | throughout | Medium | 44 px hit areas |
| F11 | Symbol glyphs as icons: `⬡ Mesh`, `▦ Lab`, `⊕ Queue`, `✎ Edit…→`, `⊕ Add`, `⊖ Remove`, `→ Push to Library`, `+ New`, `⚠` empty-state, `◐ view` | 145–146, 233, 271, 315–316, 333–334, 402, 521, 3861 | Medium | drawn SVG only |
| F12 | Hex colours everywhere, five unrelated accents (cyan, lime, green `#18f370` section labels, violet active/export, pink remove); no OKLCH tokens | 7–118, inline styles | Medium | OKLCH tokens; category tints |
| F13 | Heavy inline styles and `onmouseover` handlers on markup | 233, 251–253, 314–316, 321–382 | Low | housekeeping for stage 2 |
| F14 | `alert()` fallback for offline queue | 3879 | Low | shared header message |
| F15 | Ghost-preview checkbox is a visibility toggle that could be always on | 378–381 | Low | fewer choices; no visibility toggles |
| F16 | Duplicate readout blocks: Library and Builder each have their own stiffness panel (ids `m*` vs `bldM*`) and own radius/tiling controls | 255–301, 386–431 | Info | keep both id sets; show one per mode |
| F17 | Stiffness surface auto-spins with no Reduce Motion check | 1359–1362 | Low | respect Reduce Motion |
| F18 | Export density differs from on-screen density in Builder (export sums raw length, UI applies `beamSharingFactor`) | 3111 vs 3239–3244 | Info | not a UI fix; export must stay byte-identical, flag only |
| F19 | Comment says 14 topologies; catalog has 22 | 568 | Info | — |
| F20 | Console at 390 px run: two `CONTEXT_LOST_WEBGL: loseContext` warnings (not from beam code; likely the queue client or context limit). Zero page errors at desktop and phone | — | Low | recheck in stage 6 |

Horizontal scroll (page / control column), both tabs:

| Width | Page | Panel |
|---|---|---|
| 390 | 0 | 18 px Library, 29 px Builder |
| 700 | 0 | 0 |
| 1024 | 0 | 0 |
| 1440 | 0 | 0 |

## 3. Proposed section map

Library / Builder becomes a **source** choice inside the first section instead of a separate tab bar. Each section shows the Library or Builder variant of its controls (`data-mode` on the blocks, native ids unchanged). One section at a time on rail (desktop) and tab bar (phone).

| # | Section (icon) | Library | Builder |
|---|---|---|---|
| 01 | **Cell** (`cell`) | source segmented Library / Builder; topology dropdown; description; "Edit in Builder" | source segmented; cell name; Push to Library; New |
| 02 | **Draw** (`pen`) | short note + "Edit in Builder" button | edit Add / Remove; snap density; symmetry; Undo / Redo / Clear; status line (ghost toggle removed or kept, see decisions) |
| 03 | **Geometry** (`volume`) | radius; cell size; tiling | radius; tiling |
| 04 | **Analyze** (`analyze`) | material E<sub>s</sub>, ν<sub>s</sub>; structure tiles; stiffness surface + Zener; moduli; status | same, `bld*` ids; material shared with Library |

- Max items per group: Draw has 6 (Add/Remove, snap, symmetry, history, status, ghost). Analyze readouts stay as tiles (not controls).
- Viewer overlay: clip radio X/Y/Z (`CONFIG.clip`), view menu (`CONFIG.viewMenu`), status. Quality buttons removed by adaptive render (decision). Shader "compiling" pill folded into `#status`.
- Header `.fh-right`: help `?`, Recipe menu (Open recipe…, Export JSON, Queue, Choose queue…, Mesh, Lab), version, tool icon. LTCX and CSV: see decisions.
- Icons from the kit: `cell`, `pen`, `volume`, `analyze`, plus `recipe`, `open`, `download`, `queue`, `folder`, `layers`, `flask` in the Recipe menu, `plus`, `minus`, `trash`, `refresh` for Builder actions. Drawn icons for Push to Library and Undo/Redo arrows don't exist in the kit yet: add to liquid-ui (`undo`, `redo`, `upload`) rather than in the tool.

## 4. Control → widget map

Grep cross-check of markup (lines 119–443): `<select>` 2, `type="range"` 3, `type="number"` 2, `type="text"` 1, `type="checkbox"` 1, `<button>` 34 = **43 static controls**. Created by script: view menu (1 button, 5 checkboxes, 1 reset), help (trigger, close, 15 (i)), queue (main + caret, or fallback). No `role=switch` buttons.

**Selects (2)**

| Id | Widget | Notes |
|---|---|---|
| `#topology` | dropdown, 5 optgroups + dynamic "Custom" group | `bldPushToLibrary()` appends options at runtime: the dropdown must rebuild its groups when the select's children change (MutationObserver on `childList`); otherwise pushed cells don't appear. Kit gap: `liquidSelect` builds groups once |
| `#bldSymmetry` | dropdown | 7 options; `relabel` to "None", "Mirror X", … "Octahedral (48)" |

**Ranges (3)**

| Id | Widget | Notes |
|---|---|---|
| `#radius` | slider, `format v.toFixed(3)` | row has `<label>`; add `.sl` + `data-w-slider` |
| `#cell` | slider, `format v.toFixed(2)` | or remove if Matt agrees (F8) |
| `#bldRadius` | slider, `format v.toFixed(3)` | row has no `<label>` (uses `.sec-lbl`): stage 2 adds one; help anchor is `sec:'#bldRadius'` |

**Number / text inputs (3)**

| Id | Widget | Notes |
|---|---|---|
| `#matE` | native, restyled to 44 px | 100–2 000 000 MPa: a slider is wrong for this range; typed value is the point. Unit `MPa` shown beside |
| `#matNu` | native, restyled | −0.99–0.49, typed |
| `#bldCellName` | native text field, restyled | free text |

**Checkboxes (1 static + 5 view)**

| Id | Widget | Notes |
|---|---|---|
| `#bldGhost` | toggle, or remove (always on) | template `switches` expects `role=switch` buttons; a checkbox needs a small wire addition (mirror `checked`, dispatch `change`) |
| view menu ×5 (`.f13v-p input`) | toggles via `CONFIG.viewMenu` | template handles these |

**Segmented groups (native buttons, clicked by the widget) (7 groups, 20 buttons)**

| Group | Buttons | Widget | `current()` / sync |
|---|---|---|---|
| source | `#tabLibrary`, `#tabBuilder` | segmented (in Cell section) | `currentTab`; class `active` |
| Library tiling | `#btnTile1–3` | segmented sm | `currentTile`; class `active` |
| Builder tiling | `#bldTile1–3` | segmented sm | `Builder.tile` |
| snap density | `#btnSnapCoarse`, `#btnSnapMedium`, `#btnSnapFine` | segmented, labels "15 / 27 / 63" | `Builder.snapDensity` |
| edit mode | `#btnModeAdd`, `#btnModeRemove` | segmented with drawn `plus` / `minus` icons; Remove keeps coral tint | `Builder.mode` |
| quality | `#btnQLow/Med/High/Ultra` | removed (adaptive render) — keep hidden natives so `onQualityChange` and help text stay valid until stage 4 | `currentQuality` |
| clip | `#clipPillX/Y/Z` | radio via `CONFIG.clip` `{get:()=>clipState.axis, set:n=>onClipPill(n)}` | add `F13UI.onClipChange` hook in `updateClipPills` |

**Action buttons (14 static + 6 dynamic)**

| Selector | Widget | Notes |
|---|---|---|
| `#editCellBtn` | progress/action button (kit `progressButton` without progress) or native restyled | primary action in Cell (Library) |
| `#exportBtn`, `#bldExportBtn` | hidden; Recipe menu "Export JSON" clicks the one for the active mode | `CONFIG.recipe.exportBtn` must be mode-aware (function, not a fixed id) |
| `#exportLtcxBtn`, `#exportCsvBtn`, `#bldExportLtcxBtn`, `#bldExportCsvBtn` | Recipe menu items or native restyled row in Analyze | decision |
| `.fh-mesh-btn` Mesh, Lab (no ids) | hidden; Recipe menu Mesh / Lab | give ids in stage 2 |
| Push to Library, + New (no ids) | native restyled, drawn icons | give ids (`bldPushBtn`, `bldNewBtn`) |
| Undo, Redo, Clear (no ids) | native restyled icon buttons | give ids; Clear is destructive (Undo covers it) |
| queue `.f13ldq-main` / `.f13ldq-caret` / `#f13ldq-fallback` | Recipe menu Queue / Choose queue… | template handles |
| view `◐ view` button, `.f13v-r` reset | kit view-menu wiring | template handles |
| `.f13h-trigger`, `.f13h-close`, `.f13h-i` ×15 | stay native (help engine) | restyle to 44 px hit area |
| `#clipHandleHit` (div) | stays native | pointer drag handle; don't capture on hosts with buttons |

**Total:** 2 selects + 3 ranges + 3 typed fields + 6 checkboxes + 20 segmented buttons (7 groups) + 14 static action buttons + 6 dynamic action buttons (view, reset, queue ×2, help trigger, close) = **54 controls** (43 static + 11 script-made), plus 15 help (i) left as is. Widgets: 2 dropdowns, 3 sliders, 6 toggles (or 5 if ghost goes), 6 segmented/radio (quality dropped), 1 Recipe menu (absorbs 7 buttons). Native by design: 3 typed fields, help buttons, clip handle, Builder action buttons.

## 5. Render cost

- **Shader build:** `Raymarcher._buildShader(params)` (1637) returns the GLSL string. Structure depends only on `params.quality` (loop bound 96/160/256/512) and the constant `MAX_BEAMS=256`. Topology, Builder beams, radius, tile and clip are uniforms.
- **Recompile call sites:** `_submitShader` from `setParams` (1856) and `setBuilderParams` (1872), only when `!prev`, no program, or `quality` changed. Callers: init (3470), `onQualityChange` (2336) via `setParams` / `Builder.refresh`.
- **Slider tick recompiles? No.** Measured: 0 compiles over 30 `radius` + 30 `cell` ticks, 0 on topology change, 0 on tiling change, 2 (vertex + fragment) on a quality change. Lesson 1 is already met. Per-tick cost is CPU homogenization (F7), not compiles.
- **Quality tiers:** Low 96, Med 160 (default), High 256, Ultra 512 steps; user-picked overlay buttons. DPR capped at 1.5; canvas resized inside `render()` when CSS size changes. Soft shadows already pause while interacting (`uF13Interact`, F13LD-VIEW).
- **Loop:** continuous rAF that never stops (`_startLoop`, 2108). Main view draws only when `_dirty`, but auto-rotate is on by default and the Builder armed pulse forces redraws. Both stiffness surfaces draw every frame, hidden or not (F5). Every main frame re-uploads 2×256 vec3 uniform arrays.

**render-quality.md changes needed**

| § | Change |
|---|---|
| 1 | Step count becomes a uniform with a const upper bound (512) and `if(i>=uSteps)break;`. Quality then never recompiles; only init compiles. Assert 0 compiles per drag (already true) |
| 2 | Interactive tier: ~160 steps into `lowFB` at adaptive scale 0.4–0.85, blit LINEAR. Rest: 384 steps full-res. DPR cap 1.5 → 2 (sharper; costs more, decision). Context `antialias:false`. Keep `preserveDrawingBuffer` only if `F13LD_snapshot` needs it, else render before read |
| 3 | Rest frame in strips into `hiFB`, blit once |
| 4 | On-demand loop: `requestRender`/`kick` on uniform changes, orbit, wheel, Builder hover/arm, clip drag; loop stops when idle (assert 0 draws in 1 s, auto-rotate off). Armed-point pulse: ≤30 fps, overlay only (2D `#axisCanvas`), not the ray-march. Stiffness surfaces: redraw on `update()`, spin ≤30 fps only when on screen and the main view is idle, stop under Reduce Motion, skip the hidden mode's surface |
| — | Quality buttons removed; help item `quality` rewritten (no anchor, (i) count unaffected) |
| — | Optional: defer homogenization to `change` (release) or rAF-coalesce it during drags so ticks stay ≈0 ms |

## 6. Recipe import

- **Formats:** `beam` (Library) and `beam-builder` (Builder), both `meta.version '0.2.1'`. Recognise **both** values; never rename either; no `family` key.
- **Template gap:** the generic `check()` compares against one `TOOL.id`. Beam needs `TOOL.ids:['beam','beam-builder']` (or `TOOL.accepts(tool)`). That is a change to the generic half → make it in `liquid-ui/templates/f13ld/recipe-import.js` (optional field, default `[TOOL.id]`) so all tools stay identical. `LEGACY` already maps both to `F13LD.beam`.
- `TOOL.major`: exports say `0.2.1`, so `parseInt` gives 0. Use `major:0`; versions `1.x` get the "newer" warning.
- `looksLike(r)`: `Array.isArray(r.beams)` of 6-number arrays, and `r.geometry.radius` numeric, and `r.topology.name` or `r.cell.name`.
- `exportRecipe`: active mode (`currentTab==='builder' ? bldBuildExportData() : buildExportJSON()`), so Undo restores the mode the user was in.

**plan / apply — `beam`**

| Key | Apply via | Range |
|---|---|---|
| `topology.name` in `BEAM_CATALOGS` | `#topology.value` + `onTopologyChange()` | — |
| `topology.name` not in catalog, `beams` present (a pushed custom cell) | register under the **exact** key (refactor `bldPushToLibrary` into `registerCustomCell(key, name, beams)`), then select it | ≤256 beams, coords clamped to [−1,1] |
| `geometry.radius` | `#radius.value` + `input` event | 0.02–0.30, step 0.005 |
| `geometry.cell` | `#cell.value` + `input` | 0.5–3.0, step 0.05 |
| `homogenization.material.Es_MPa`, `.nu_s` | `#matE`, `#matNu` + `onMatChange()` | 100–2e6; −0.99–0.49 |
| then | `switchTab('library')`, one `updateMetrics()` | |

**plan / apply — `beam-builder`**

| Key | Apply via | Range |
|---|---|---|
| `beams` | `Builder.loadBeams()` (clears history) | ≤256, coords clamped |
| `cell.name` | `#bldCellName.value`, `Builder.cellName` | string |
| `geometry.radius` | `#bldRadius.value` + `bldSetRadius()` | 0.02–0.30 |
| `geometry.cell` | ignored (always 1.5) | |
| material | as above | |
| then | `switchTab('builder')` (calls `Builder.refresh()` once) | |

- Reference: the existing `?b=` handler (3446–3467) does the Builder half already.
- **Excluded (not restored):** `metrics.*`, `topology.beam_count`, `cell.beam_count`, every `homogenization` result (`counts`, `stiffness_C_MPa`, `compliance_S_MPa_inv`, `engineering_constants_MPa`, `poisson_ratios`, `anisotropy`, `valid`, `reason`, `method`, `voigt_convention`). They recompute.
- **Material is an input stored inside `homogenization`.** Restoring it is needed for a clean round-trip (homogenization output depends on it). See decisions.
- **Setters available:** `onTopologyChange`, `onParamChange(input,id)`, `onTileChange`, `onMatChange`, `switchTab`, `Builder.loadBeams`, `Builder.refresh`, `bldSetRadius`, `bldSetSnap`, `bldSetSymmetry`, `bldSetMode`, `bldSetTile`. Missing: `registerCustomCell` (factor out of `bldPushToLibrary`).
- **Round-trip test:** random states per mode (topology incl. custom, radius, cell, material; Builder beams from random snap pairs + symmetry, name, radius). Ignore `timestamp`. Tiling, snap, symmetry and ghost aren't exported, so they are not part of the round-trip.
- **Undo caveat:** importing a `beam-builder` recipe overwrites Builder beams. Undo from a Library snapshot restores Library but not the previous Builder beams. Acceptable, or snapshot both modes (decision not needed unless Matt wants it).

## 7. Help system

- `SECTIONS` has **19 items**, 15 with anchors → **15 (i)** in the DOM (measured). Unanchored: `export`, `history`, `bldexport`, `quality`.
- Engine config: `ROW_SEL='.sc-group'`; placement modes used: `after`, `sec`, `row`.
- Anchors that must survive:
  - `after:'#tabBuilder'` — (i) is inserted after the tab button. If the tab bar becomes a hidden native, this (i) disappears from view: re-anchor to the source segmented host (count unchanged).
  - `sec:` `#topology`, `#stiffSurfCv`, `#bldCellName`, `#btnSnapCoarse`, `#btnModeAdd`, `#bldRadius`, `#bldSymmetry`, `#bldTile1` — `secLabelFor` walks **previous siblings up the parent chain** to the nearest `.sec-lbl`. Every such control must keep its `.sec-lbl` as a preceding sibling of itself or an ancestor, inside its new section.
  - `row:` `#radius`, `#cell`, `#btnTile1`, `#matE`, `#matNu` — need the `.sc-group` row with its `<label>`. `#bldGhost` resolves through the wrapping `<label>`.
- Classes/ids: `.f13h-trigger`, `.f13h-i`, `.f13h-drawer`, `.f13h-scrim`, `.f13h-close`, `style#f13h-theme` vars (`--f13h-accent` is cyan `#22d3ee`: switch to the confirmed accent).
- Add "show" buttons (`CONFIG.helpTargets`) for the 15 anchored items; `tabs` → source segmented, `stiffness` → `#stiffSurfCv` (or `#bldStiffSurfCv` in Builder).

## 8. Accent colour (to confirm with Matt)

| Colour | Hex | OKLCH | Used for |
|---|---|---|---|
| cyan | `#22d3ee` | 0.797 0.134 211.5 | slider thumbs, quality active, metric values, help accent, view checkboxes, Add, shader dot, clip X |
| lime | `#c8f542` | 0.908 0.201 123.0 | active tab, Edit in Builder, Push/New, brand |
| beam family olive | `#b8cf50` | 0.813 0.153 118.7 | rendered beams (shader base colour, line 1772) |
| tool mark | `#4A5A0F` / `#ECF1DA` | 0.439 0.098 121.4 | tool name, header icon |
| green | `#18f370` | 0.843 0.232 149.3 | section labels |
| violet | `#7c6ff7` | 0.623 0.196 283.4 | active mode button, export buttons |
| pink | `#ff6b9a` | 0.723 0.185 3.1 | Remove, negative Maxwell |

**Recommendation: aqua, `oklch(0.84 0.14 200)`** (kit `Liquid.ACCENTS.aqua`), the cleaned-up version of beam's cyan. It is already beam's most-used interactive colour, it separates beam from TPMS lime, and the olive beams read well against it.
- Catch 1: the clip X axis is also cyan (`#22d3ee`). With an aqua accent, the X clip bead would match the accent. The kit's `axisX` accent stays as is: acceptable, since the clip radio is an axis colour, not an action.
- Catch 2: noise, grain and bundle also list cyan. If Matt wants every tool distinct, beam's alternative is the family olive raised to kit lightness, `oklch(0.84 0.16 118)`, but that sits close to TPMS lime (hue 123–125).

## 9. Risks

- **Two export paths in one Recipe menu.** Export JSON, Queue, Mesh and Lab must follow the active mode. Today they don't (F4), so fixing it changes what Mesh/Lab receive from the Builder tab. Payloads per function stay byte-identical.
- **Dynamic topology options.** Pushed cells add an optgroup at runtime. The dropdown widget must rebuild on `childList` changes, or pushed cells are unreachable.
- **Help anchors on restructure.** The `sec:` anchors depend on sibling order. Moving a control out from under its `.sec-lbl` silently drops its (i). The `#tabBuilder` (i) goes invisible if the tab bar is hidden.
- **Generic recipe-import change.** Accepting two `meta.tool` values needs a small change to the "identical across tools" half. Make it in liquid-ui first so every tool vendors the same file.
- **Mode-split sections.** Library and Builder keep separate id sets (`m*`/`bldM*`, `#radius`/`#bldRadius`, `#btnTile*`/`#bldTile*`). App code writes to both; hiding by mode must never remove either.
- **WebGL contexts.** Main view + 2 stiffness surfaces + the kit's shared context = 4. Fine on desktop. On phones, lazy-create the Builder surface (already lazy) and never draw the hidden one.
- **Homogenization jank on drag** (F7). Up to 42 ms per tick for dense Builder cells in swiftshader. Widgets won't fix it; stage 4 should coalesce it.

## Decisions for Matt

- **Accent:** aqua `oklch(0.84 0.14 200)` (recommended), or olive `oklch(0.84 0.16 118)`?
- **Library / Builder:** turn the tab bar into a Library / Builder choice inside the first section ("Cell"), as proposed? Sections: Cell, Draw, Geometry, Analyze.
- **Recipe menu follows the mode:** on the Builder tab, Export JSON / Queue / Mesh / Lab send the Builder cell (`beam-builder`). Today they send the Library cell. OK to fix?
- **LTCX and CSV:** extra items in the Recipe menu, or a small export row at the end of Analyze?
- **Quality buttons:** remove them (adaptive render replaces them)?
- **Sharper rest frame:** raise the pixel-ratio cap from 1.5 to 2 (TPMS standard)? It costs more GPU time at rest.
- **Ghost preview checkbox:** remove it (always on), or keep it as a toggle?
- **Cell size slider:** it changes only export/LTCX/CSV, nothing on screen. Keep it, move it next to export, or keep it as is?
- **Material on import:** restore E<sub>s</sub> and ν<sub>s</sub> from a recipe's `homogenization.material`? They're inputs, but they sit inside the results block that's otherwise skipped.
