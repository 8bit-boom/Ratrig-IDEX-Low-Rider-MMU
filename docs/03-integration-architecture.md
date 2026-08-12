# Why T2-T5 (not T0-T3) for the MMU gates, and how the integration works

## The collision, and why the obvious fix doesn't work

Both stock RatOS IDEX and stock Happy Hare want to define G-code macros named
`T0`, `T1`, `T2`, ...:

- **RatOS IDEX** ships `[gcode_macro T0]` / `[gcode_macro T1]` meaning "select
  *physical carriage* 0 (left toolhead)" / "*physical carriage* 1 (right
  toolhead)".
- **Happy Hare**, with `mmu_num_gates: 4`, auto-generates `[gcode_macro T0]` ..
  `[gcode_macro T3]` meaning "select *logical gate* 0-3 on the MMU".

The obvious fix is "just renumber one of them" — e.g. let Happy Hare's gates be
`T0`-`T3` and move the standalone left toolhead to a new number. **This doesn't
work**, because `T0`/`T1` in RatOS are not just slicer-facing entry points — many
*other* RatOS macros hardcode lookups like `printer["gcode_macro T0"].parking_position`
or `printer["gcode_macro T%s" % dual_carriage_index]` (`dual_carriage_index` is
always physically `0` or `1`, one per carriage — this is *hardware* indexing, not
tool/color indexing). This is used by, among others:

- `PARK_TOOLHEAD` (runs at print end/cancel, and during pauses)
- `_IDEX_SINGLE` / `IDEX_COPY` / `IDEX_MIRROR`
- `JOIN_SPOOLS` / `_JOIN_SPOOL`
- `GENERATE_SHAPER_GRAPHS` / `MEASURE_COREXY_BELT_TENSION`

Each reads `variable_parking_position`, `variable_purge_after_load`,
`variable_active`, `variable_has_oozeguard`, filament-sensor names
(`toolhead_filament_sensor_t0`/`_t1`), etc. **directly off the macro objects
literally named `T0` and `T1`.** If we repoint `T0`/`T1` at MMU gates, those
variables disappear and every one of the macros above breaks the next time it
runs — most importantly `PARK_TOOLHEAD`, which fires on essentially every print.

So `T0` and `T1` must keep meaning exactly what RatOS ships: physical carriage 0
(left) and physical carriage 1 (right). This repo does not touch them at all.

## The resolution used in this repo

Give the 4 MMU gates **new, non-colliding macro names — `T2`-`T5`** — and leave
`T0`/`T1` completely untouched.

| Slicer tool | Macro | Meaning | Physical result |
|---|---|---|---|
| T0 | RatOS stock `T0` (untouched) | left toolhead, standalone | `extruder`, no MMU |
| T1 | RatOS stock `T1` (untouched) | right toolhead, "as-is" | `extruder1`, whatever gate is currently loaded — not normally used directly in MMU prints, see note below |
| T2 | our new `T2` | MMU gate 0 | right toolhead, `extruder1`, gate 0 |
| T3 | our new `T3` | MMU gate 1 | right toolhead, `extruder1`, gate 1 |
| T4 | our new `T4` | MMU gate 2 | right toolhead, `extruder1`, gate 2 |
| T5 | our new `T5` | MMU gate 3 | right toolhead, `extruder1`, gate 3 |

`config/macros/idex_mmu_integration.cfg` implements `T2`-`T5`:

- Each calls a small wrapper, `_MMU_ENSURE_RIGHT_CARRIAGE`, **then**
  `MMU_CHANGE_TOOL TOOL=<gate 0-3>`. The wrapper checks
  `printer["dual_carriage"].carriage_1` and, only if the right carriage isn't
  already primary, calls RatOS's own `_SELECT_TOOL T=1` — using RatOS's real
  parking/offset/input-shaper logic, not a reimplementation. If the right
  carriage is already active (the common case — you're mid-print doing color
  changes), this is a no-op and the gate change is as fast as a normal Happy Hare
  toolchange.
- Nothing named `T0` or `T1` is declared anywhere in this repo.

### Required one-time manual step after installing Happy Hare

Happy Hare's installer auto-generates `[gcode_macro T0]` .. `[gcode_macro T3]` in
your `mmu/mmu_macro_vars.cfg` (each just calling `MMU_CHANGE_TOOL TOOL=n`). Those
**would** collide with RatOS's real `T0`/`T1` if left in place (whichever file is
included last wins, and either way something breaks). After every fresh Happy Hare
install *and every Happy Hare update that touches this file*, open
`~/printer_data/config/mmu/mmu_macro_vars.cfg`, find the generated
`[gcode_macro T0]` through `[gcode_macro T3]` blocks (look for the `MMU_CHANGE_TOOL
TOOL=` line inside each), and delete or comment out all four. `config/macros/
idex_mmu_integration.cfg` supplies the replacements (`T2`-`T5`) — you're not losing
functionality, just removing the colliding auto-generated wrapper names. Re-check
this after any Happy Hare upgrade, since the installer may regenerate the file.

## Using T1 directly

`T1` still works exactly as RatOS ships it — it activates the right carriage
without changing which MMU gate is loaded. It's useful for jogging/testing the
right toolhead by hand. Slicers must number extruders contiguously from 0, so a
6-toolhead-wide slicer profile (`docs/04-slicer-setup.md`) will include a slot for
`T1` even though a pure MMU multi-material project typically never emits it —
that's expected and harmless; just don't assign any filament/color to that slot in
your project.

## If your gate count isn't 4

Change `mmu_num_gates` in `mmu_parameters_lowrider.cfg`, and add/remove the
corresponding `T<n>` macro block(s) in `idex_mmu_integration.cfg` and gate-stepper
sections in `mmu_hardware_lowrider.cfg`. The MMU gate macros always start at `T2`
and run for `num_gates` slots (e.g. 5 gates → `T2`-`T6`). Update the slicer's
extruder count to match (`docs/04-slicer-setup.md`).
