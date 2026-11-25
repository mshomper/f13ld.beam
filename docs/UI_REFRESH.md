# F13LD.beam UI refresh log

| Stage | Date | Branch | Commit | Checks | Open decisions |
|---|---|---|---|---|---|
| 1 Audit | 2026-11-25 | `ui-refresh` | none yet | `docs/UI_AUDIT.md` + 11 screenshots in `docs/ui-audit/`; 0 page errors; 0 compiles per slider drag; panel scrolls sideways at 390 px (18/29 px); 15 help (i) | accent (aqua vs olive); tab bar → source choice in Cell section; Recipe menu follows Library/Builder mode; LTCX/CSV placement; remove quality buttons; DPR cap 1.5 → 2; ghost toggle; cell size slider; restore material on import |

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
