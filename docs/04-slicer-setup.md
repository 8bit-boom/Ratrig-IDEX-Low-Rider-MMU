# Slicer setup (OrcaSlicer / PrusaSlicer / SuperSlicer)

Slicers number extruders contiguously starting at 0 and emit `Tn` for extruder
index `n` — you can't skip a number. Because `T1` must stay reserved for RatOS's
own "right toolhead, as-is" macro (see `docs/03-integration-architecture.md`), the
printer needs to look like a **6-extruder** machine:

| Slicer extruder index | G-code | Use it for |
|---|---|---|
| 0 | `T0` | Standalone left toolhead (its own material) |
| 1 | `T1` | **Leave unused in every project.** Reserved by RatOS; don't assign filament/color to it. |
| 2 | `T2` | MMU gate 0 |
| 3 | `T3` | MMU gate 1 |
| 4 | `T4` | MMU gate 2 |
| 5 | `T5` | MMU gate 3 |

## Printer settings

1. Set **Extruders: 6**.
2. Extruder/nozzle offsets:
   - Extruder 0 (`T0`, standalone): the left toolhead's calibrated IDEX offset.
   - Extruder 1 (`T1`, unused): doesn't matter, but for sanity set it identical to
     extruders 2-5.
   - Extruders 2-5 (`T2`-`T5`, the MMU gates): **identical** X/Y/Z offsets — they
     all print out of the same physical nozzle (the right toolhead). Use the right
     toolhead's calibrated IDEX offset (from your VAOC/manual dual-carriage
     calibration).
3. **Wipe tower**: if you use one, position it within reach of the **right**
   toolhead only (that's the only carriage that ever does a multi-material
   sequence). Left-toolhead-only (`T0`) prints don't need a wipe tower.
4. **Tool change G-code**: leave it empty / default (`Tn`). Don't add manual
   `SELECT_TOOL`/`ACTIVATE_EXTRUDER` G-code in the toolchange box — `T0` and
   `T2`-`T5` already do everything needed server-side (see
   `docs/03-integration-architecture.md`). Never assign anything to extruder
   slot 1 in a project (it's a reserved gap, not a usable material slot).

## Filament / material assignment

Assign your 4 MMU materials to extruders 2-5, and your standalone material to
extruder 0. If a given job only uses one group (e.g. a pure single-color print on
just the left toolhead, or a pure multi-material print on just the right
toolhead), leave the unused extruder(s) out of that project — RatOS and Happy Hare
don't require every tool to be used every print.

## Start / End G-code

Keep using RatOS's `START_PRINT`/`END_PRINT` macro calls as usual (they're
IDEX-aware already). Don't add MMU-specific bootstrap G-code to the slicer's start
script — Happy Hare's own print-start/print-end hooks (configured in
`mmu_macro_vars.cfg`, `_MMU_SOFTWARE_VARS`) handle homing the MMU, auto-loading the
first tool, and end-of-print unload/park. Verify those hooks are enabled
(check your installed `mmu_macro_vars.cfg` for the current key names — they've
changed across Happy Hare releases) rather than duplicating that logic in the
slicer.

## First test print

Before a full multi-material job, print something trivial that only exercises the
new numbering:

1. A single-color file assigned entirely to extruder 0 (`T0`) — confirms the
   standalone left toolhead is unaffected by any of this.
2. A single-color file assigned entirely to extruder 2 (`T2`) — confirms `T2`
   correctly swaps to the right carriage and loads MMU gate 0.
3. A short multi-color file using extruders 2-4 (`T2`-`T4`) only (no `T0`) —
   confirms in-place gate switching without carriage swaps.
4. Only once 1-3 pass, try a file mixing `T0` with `T2`-`T5`.
