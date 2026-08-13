# Hardware overview & wiring assumptions

This doc was rewritten against GcodeGearhead's official **Low Rider MMU V2.0 Build
Guide** (PDF, supplied directly by the user — printables.com itself was unreachable
from this environment, but the guide gave everything needed). Anything below
credited to "the build guide" is a real value/photo from that document, not a
guess.

## Printer side (RatRig V-Core 4.1 IDEX)

RatOS's `V-Core 4.1 IDEX` printer profile (`Rat-OS/RatOS-configuration`, printer
folder `printers/v-core-4-1-idex`) gives you, out of the box:

- Kinematics: `hybrid-corexy-idex` (`ratos_hybrid_corexy`, `inverted: true`)
- `[dual_carriage]` on the X axis — carriage 0 = left toolhead (`extruder`),
  carriage 1 = right toolhead (`extruder1`)
- Two independent toolboards (default RatOS BOM: BTT EBB42-12 CAN toolboards, one
  per toolhead), each with its own extruder stepper, hotend, part-cooling fan,
  X-endstop, and probe
- `T0` / `T1` G-code macros that call the internal `_SELECT_TOOL` macro to park the
  inactive carriage and activate the requested one

**This repo's assumption:** the Low Rider MMU's output feeds **only the right
toolhead (carriage 1 / `extruder1`)**. The left toolhead (carriage 0 / `extruder`)
is wired and configured exactly as a stock RatOS IDEX toolhead — nothing in this
repo touches its hardware config.

## MMU side (Low Rider MMU v2.0) — corrected mechanism

**Correction from an earlier version of this doc:** the Low Rider MMU is a
**Type-A** MMU in Happy Hare's terminology, not Type-B. The build guide states
this explicitly: *"The Low Rider MMU is a TYPE A MMU, this means there are:
Pregate sensors, A single drive gear, A single servo controlled selector."*

That means, unlike ERCF/Tradrack's linear selector, or Box Turtle/3MS's one
stepper per gate:

- **One NEMA17 stepper** drives filament for whichever gate is currently
  selected (shared across all gates, via an 80T gear + GT2 belt train).
- **One MG996R servo** rotates a CAM shaft that mechanically presses the selected
  gate's lever/BMG gear into that shared drive train — this is the "selector."
  Happy Hare's config name for it is `selector_type: ServoSelector`.
- **Six pre-gate limit switches** (D2FC NO+NC), one per lane, detect filament
  entering each gate and drive gate autoload.
- **No filament encoder, and no shared gate sensor.** The build guide is explicit:
  *"In this version of the MMU we do not run a gate sensor as we load filament up
  to the extruder sensor and then park it just before."* Gate parking/homing is
  done using the **extruder entry sensor on the toolhead**
  (`gate_homing_endstop: extruder`), not a sensor local to the MMU.
- **Six WS2812 (Neo Pixel GRBW) LEDs**, one per gate, daisy-chained.

This repo's Happy Hare config (`config/happy-hare/`) now reflects this: one
`[stepper_mmu_gear]`, one `[mmu_servo selector_servo]`, no
`stepper_mmu_gear_1/2/3` and no `[mmu_encoder]`.

## Electronics — real BOM from the build guide

- **Controller: a single BTT EBB42 v1.2 CAN toolboard** (no Max31865), mounted on
  the MMU itself and connected to the printer's CAN bus — not a separate/dedicated
  MCU, and not one-per-gate. Flash Klipper to it (Katapult recommended) before
  running the Happy Hare installer.
- **LM2596 buck converter**, stepping the 24V MMU supply (fed *separately* from
  the printer PSU, not off the toolboard's own input) down to 5.1V for the
  pre-gate switch logic level.
- Servo (MG996R) and gear stepper (NEMA17) run directly off the EBB42.
- The build guide's own wiring diagram gives this exact pin-out for their
  reference build (verify against your own soldering if you deviated):

  | Signal | EBB42 pin |
  |---|---|
  | Gear stepper UART | PA15 |
  | Gear stepper STEP | PD0 |
  | Gear stepper DIR | PD1 |
  | Gear stepper ENABLE | PD2 |
  | Selector servo signal | PA3 |
  | Neopixel data | PD3 |
  | Pre-gate switch, gate 0 | PB6 |
  | Pre-gate switch, gate 1 | PB5 |
  | Pre-gate switch, gate 2 | PB7 |
  | Pre-gate switch, gate 3 | PB8 |
  | Pre-gate switch, gate 4 | PB9 |
  | Pre-gate switch, gate 5 | PB4 |

  For a **4-gate build** (this repo's default, per your earlier answer) you only
  need gates 0-3 → `PB6, PB5, PB7, PB8`. Leave `PB9`/`PB4` unused.

- Gear stepper current: build guide's own guidance — **run current = 0.85 x the
  motor's rated current, capped at 1A** (the EBB42's driver limit).

## What's still genuinely build-specific (not guessable)

- **Extruder entry sensor / toolhead sensor pins** — these live on the **right
  toolhead's own board** (its EBB42, per RatOS's stock IDEX BOM), not the MMU's
  board. The build guide's own example uses their personal toolboard alias
  (`XOL:PB8` / `XOL:PB9`) which only makes sense for their machine. You must
  substitute your right toolhead's actual sensor pin names (or aliases) here —
  see `mmu_hardware_lowrider.cfg`.
- **Servo gate angles.** The guide gives a starting point
  (`26,58,90,118,147,180` for their 6-gate reference build) but is explicit that
  these must be tuned per-machine (assembly tolerances, and there were two CAM
  generations with different belt lengths — 144mm vs 146mm). Calibrate with
  `SET_SERVO`/`MMU_SELECT` per `docs/05-calibration.md`; don't trust the numbers
  as-is.
- **`gate_homing_max` / `gate_parking_distance`.** The example build's values
  (1310mm / 1200mm) reflect *their* tube routing distance from gate to toolhead
  sensor, which depends entirely on where you physically mount the MMU relative
  to the right toolhead. Measure your own.
- **Filament cutter.** The build guide's author has an optional toolhead-mounted
  filament cutter and configures `form_tip_macro: _MMU_CUT_TIP` with
  machine-specific cutter coordinates. The Low Rider MMU's core BOM does not
  require a cutter — if you don't have one, use Happy Hare's default
  `_MMU_FORM_TIP` instead and ignore the cutter-specific settings.
