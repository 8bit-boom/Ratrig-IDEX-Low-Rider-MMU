# Calibration order

Do these in order. Each stage assumes the previous one is solid — don't move on
until the current stage behaves correctly.

## Stage 1 — Printer, no MMU involved

1. `Config_checks.html`-style sanity check (Klipper docs), PID tune both hotends
   and the bed.
2. Home, then run `GENERATE_SHAPER_GRAPHS` for both toolheads (RatOS disables
   plain `SHAPER_CALIBRATE` on IDEX machines on purpose — it's unreliable with two
   independent carriages; use the graph-based flow and hand-enter
   `variable_shaper_*_freq`/`variable_shaper_*_type` per RatOS's own guidance in
   `macros/idex/idex_is.cfg`).
3. Calibrate IDEX toolhead-to-toolhead XY offset (VAOC if fitted, or RatOS's manual
   dual-carriage offset procedure) and Z offset for both toolheads independently.
4. Pressure advance for both extruders separately.
5. Print a plain single-color part on `T0` (left) and one on the right toolhead
   using a normal, non-MMU direct-load (temporarily bypass the MMU by hand-feeding
   filament into the right toolhead's extruder, if you want to isolate "is the
   right toolhead itself okay" from "is the MMU path okay"). Confirm both toolheads
   individually print clean before wiring the MMU into the mix.

## Stage 2 — MMU hardware, standalone (Happy Hare's own calibration flow)

With the printer otherwise idle (no active print):

1. `MMU_STATUS` — confirms Happy Hare loaded cleanly with your `mmu_hardware_lowrider.cfg`
   pins.
2. Per-gate gear rotation distance calibration
   (`MMU_CALIBRATE_GEAR GATE=0` .. `GATE=3`, or the Type-B bulk flow — see Happy
   Hare's wiki page **MMU Calibration TypeB**, which covers exactly this
   individual-drive-stepper-per-gate class of MMU that the Low Rider MMU belongs
   to).
3. Bowden length calibration (`MMU_CALIBRATE_BOWDEN`) — from the gate merge point
   to the right toolhead's extruder entrance.
4. Gate/sensor sanity: manually feed filament into each gate and confirm the
   pre-gate sensor(s) and gate/exit sensor (if fitted) report correctly in
   `MMU_STATUS` / the Mainsail MMU panel.
5. Toolhead sensor calibration on the right toolhead (`MMU_CALIBRATE_TOOLHEAD` or
   equivalent for your Happy Hare version) — needed for reliable load/unload
   endpoint detection.
6. Run `MMU_TEST_LOAD`/`MMU_TEST_UNLOAD` (or your Happy Hare version's equivalent
   test commands) gate by gate, watching the physical filament path the whole way,
   before trusting it during an unattended print.

Do all of this with the printer's **right carriage parked at rest** — you don't
need `T0`-`T3` (our wrapped macros) for pure MMU-side calibration; use Happy Hare's
raw `MMU_...` commands directly so you're not also exercising the IDEX carriage
logic at the same time.

## Stage 3 — Combined

1. Home, then issue our `T2` (should swap to the right carriage *and* load MMU
   gate 0). Confirm the carriage actually moved and filament actually loaded.
2. From `T2`, issue `T3`, `T4`, `T5` in turn — these should **not** move the
   carriage (already on the right side), only swap the gate.
3. From any of `T2`-`T5`, issue `T0` — confirms the carriage parks the right
   toolhead and switches to the left one. Then issue `T2` again — confirms the
   round trip back.
4. Run the three short test prints listed at the end of `docs/04-slicer-setup.md`.
5. Only after all of the above is reliable, attempt a full mixed `T0` + `T2`-`T5`
   print.

## If something in Stage 3 misbehaves

Almost always it's one of:

- Happy Hare's auto-generated `[gcode_macro T0]`-`[gcode_macro T3]` block in
  `mmu_macro_vars.cfg` was never deleted (see the "Required one-time manual step"
  in `docs/03-integration-architecture.md`) — it's silently overriding, or being
  overridden by, RatOS's real `T0`/`T1`. Check `RESTART` console output for macro
  redefinition warnings.
- `extruder: extruder1` missing/wrong in `mmu_parameters_lowrider.cfg` — Happy Hare
  will happily calibrate and run against the wrong (left) extruder if this isn't
  set, and everything will look fine until you check which nozzle actually
  extruded.
- `config/macros/idex_mmu_integration.cfg` (defining `T2`-`T5`) isn't included, or
  is included before Happy Hare's own MMU config, so `MMU_CHANGE_TOOL` isn't ready
  yet when it's referenced.
