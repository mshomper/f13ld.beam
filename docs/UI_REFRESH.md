# F13LD.beam UI refresh log

| Stage | Date | Branch | Commit | Checks | Open decisions |
|---|---|---|---|---|---|
| 1 Audit | 2026-11-25 | `ui-refresh` | none yet | `docs/UI_AUDIT.md` + 11 screenshots in `docs/ui-audit/`; 0 page errors; 0 compiles per slider drag; panel scrolls sideways at 390 px (18/29 px); 15 help (i) | accent (aqua vs olive); tab bar → source choice in Cell section; Recipe menu follows Library/Builder mode; LTCX/CSV placement; remove quality buttons; DPR cap 1.5 → 2; ghost toggle; cell size slider; restore material on import |
| 2 Layout shell | 2026-11-25 | `ui-refresh` | `fd72991` | 0 page errors (desktop 1440, phone 390, all section/mode combos); horizontal scroll 0 px in 32 of 32 cells (4 widths × Library/Builder × 4 sections; page, control column, section, header); text: 0 elements under 11 px; targets: 0 under 44 px (queue caret hit box widened); export JSON byte-identical to `main` in 16 of 16 comparisons (8 Library, 6 Builder, 2 real downloads; timestamp ignored); Mesh / Lab send `beam` on Library and `beam-builder` on Builder (4 of 4); help (i) 15 → 14 (ghost removed), cards 19 → 18; help "show" lands on its control 18 of 18 on phone, 18 of 18 on desktop (one needed a longer wait for the smooth scroll); 16 screenshots + 1 peek in `docs/ui-audit/stage2-*.png` | accent kept at `oklch(0.80 0.15 121)` (reads clearly darker and less neon than TPMS lime side by side); widgets, recipe menu, render quality still native (stages 3–5); stiffness surface and quality buttons keep their old behaviour |

Stage 2 notes:
- **Analysis is live, not stale.** Beam recomputes homogenization on every change, and both exports compute it fresh, so nothing stale is exported. The one real gap was fixed instead: after a material or Builder change, returning to Library showed old Library results. `switchTab('library')` now recomputes.
- **Help anchors:** the `sec:` anchors moved to `row:` (each control has its `.sc-group` row with a label); `tabs` anchors on the source row. Section heading `directional stiffness` keeps the `.sec-lbl` class for the `stiffness` card.
- **Removed:** ghost checkbox (help card `ghost` removed with it), tab bar, shader pill (folded into `#status`), `bldRedraw()`.
- **Header:** drawn icons on Mesh / Lab, queue label without the glyph, `alert()` fallback replaced by the shared header message, `#w-recipe` mount added.
- **Layout:** stiffness readouts stack (surface on top, structure and moduli tiles side by side) because the old three-column row did not fit the 324 px column.
- **Mount points for stage 3:** `#w-source`, `#w-topology`, `#w-edit`, `#w-snap`, `#w-sym`, `#w-tile`, `#w-bldtile`, `#w-clip`, `#w-recipe`; `data-w-slider` on `radius`, `bldRadius`, `cell`; `data-native` on the native groups. The `.sec-lbl`-based help lookup is no longer needed by any anchor.
- **Open:** the viewer canvas is larger than before (full pane height), so ray-march cost per frame is higher until stage 4; quality buttons are still the old four; `#clipHandleHit` is now 44 px.

## Decisions (Matt, 2026-11-25)

Shared, all wave 1 tools:
- **Quality:** automatic, as in TPMS. Remove the manual quality control (buttons or slider); its help (i) goes with it.
- **Stale analysis:** clear homogenization / stiffness results when the design changes, and on import. Exports carry results only for the current design.
- **Missing help cards:** add them; the (i) count rises by that number (record the new count).
- **Field sliders that re-bake:** show a low-res preview bake while dragging, full bake on release. Keep it only if it measures faster to first change than today and does not slow the full bake.
- **Accent:** derived from the tool colour in the F13LD brand guidelines v2.0 (hue kept, lifted to L 0.80 for the dark UI). Final value pending Matt.

This tool:
- **Accent (proposed):** olive-lime `oklch(0.80 0.15 121)` ≈ `#afcc54`, from brand stroke `#4A5A0F`. Close to the brand neon `#c8f542` used by TPMS; lower chroma if it reads the same.
- **Quality buttons:** removed (automatic). Pixel-ratio cap to 2 comes with the render-quality pattern.

- Library / Builder becomes a choice inside Cell. Recipe menu (Export JSON / Queue / Mesh / Lab) sends the Builder cell on Builder. LTCX and CSV in the Recipe menu. DPR cap 2. Ghost preview always on (checkbox removed). Cell size slider moves next to export. Import restores E_s / ν_s.

Stage 1 approved by Matt 2026-11-25 ("use your picks").
