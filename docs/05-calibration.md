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

## Stage 2 — MMU hardware, standalone (the build guide's own calibration flow)

With the printer otherwise idle (no active print), and the right carriage parked
at rest — you don't need `T2`-`T5` (our wrapped macros) for pure MMU-side
calibration; use Happy Hare's raw `MMU_...` commands directly so you're not also
exercising the IDEX carriage logic at the same time.

This is a **Type-A, servo-selector** MMU (one shared gear stepper, one CAM
selector), so its calibration flow is the ERCF-style **MMU Calibration TypeA**
procedure, not the per-gate Type-B one. Follow it in this order — it's the exact
sequence from GcodeGearhead's build guide:

1. `MMU_STATUS` — confirms Happy Hare loaded cleanly with your
   `mmu_pins_lowrider.cfg` / `mmu_hardware_lowrider.cfg`.
2. **Servo/CAM angle calibration** — do this before anything else, the gear
   calibration below depends on the selector actually landing on the right gate:
   - `SET_SERVO servo=selector_servo angle=0` and physically check gate 0's CAM
     ear points straight down and grips the filament channel. If not, pull the
     5mm rod to disengage the CAM from the belt, rotate it into position, and
     reassemble (see the build guide's Final Assembly section/photos).
   - For each gate, `MMU_SELECT GATE=<n>` then `SET_SERVO servo=selector_servo
     angle=<n>` sweeping through candidate angles (the guide's own starting point
     for a 4-gate build was `26, 58, 90, 118` for gates 0-3) until each gate's
     CAM ear sits fully down. Nudge angles individually if any gate is slightly
     off.
   - Write the final, confirmed angles into `selector_gate_angles` in
     `mmu_parameters_lowrider.cfg`.
   - If you get `"Operation not possible. MMU has filament loaded"` here, you
     forgot to comment out `gate_switch_pin` in `mmu_hardware_lowrider.cfg` (this
     MMU doesn't have that sensor — see `docs/01-hardware-overview.md`).
3. **Gear rotation distance** — since there's only one shared gear, you calibrate
   it once and it applies to every gate:
   - Feed a piece of filament (>200mm) into gate 0 until it just protrudes past
     the ECAS fitting; mark it with a pencil.
   - `MMU_SELECT GATE=0`, then `MMU_TEST_MOVE MOVE=100`, then measure the actual
     length fed and run `MMU_CALIBRATE_GEAR MEASURED=<your measurement>`.
   - Repeat `MMU_SELECT GATE=1` → `MMU_CALIBRATE_GEAR MEASURED=<same
     measurement>`, and so on for every gate you built — you're not
     recalibrating the gear each time, just confirming the same shared stepper
     performs consistently once the servo has switched gates.
4. **Load/unload sanity**: feed filament by hand into each gate's pre-gate
   sensor; it should trigger MMU auto-load, drive the filament all the way to
   the right toolhead's extruder sensor, then retract and park just before it.
   Watch the whole physical path before trusting it unattended.
5. **Toolhead sensor calibration** — `MMU_CALIBRATE_TOOLHEAD` (requires the
   toolhead sensor from `docs/01-hardware-overview.md`), used to derive
   `toolhead_extruder_to_nozzle` / `toolhead_sensor_to_nozzle`.

Reference: `https://github.com/moggieuk/Happy-Hare/wiki/MMU-Calibration-TypeA`.

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
