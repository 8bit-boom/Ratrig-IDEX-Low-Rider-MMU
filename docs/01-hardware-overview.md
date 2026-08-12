# Hardware overview & wiring assumptions

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

**This repo's assumption:** the Low Rider MMU's Bowden output feeds **only the
right toolhead (carriage 1 / `extruder1`)**. The left toolhead (carriage 0 /
`extruder`) is wired and configured exactly as a stock RatOS IDEX toolhead — nothing
in this repo touches its hardware config.

If your build feeds the MMU into the *left* toolhead instead, swap `extruder1`/
`CARRIAGE=1` for `extruder`/`CARRIAGE=0` throughout `config/happy-hare/` and
`config/macros/idex_mmu_integration.cfg`.

## MMU side (Low Rider MMU v2.0)

The Low Rider MMU v2.0 is a **Type-B** design in Happy Hare's terminology: each gate
has its own **independent gear/drive stepper** (a "dual-gear" pinch-and-drive pair
per the vendor's own description) that grips and always holds the filament — there
is no moving **selector** stage to home or align, unlike ERCF/Tradrack-style
(Type-A) units. That is the whole point of the design ("less moving parts to align
and tinker with").

This repo therefore configures Happy Hare with:

```
mmu_vendor: Other
mmu_version: 1.0
selector_type: VirtualSelector      # no physical selector — matches BoxTurtle/3MS/NightOwl class
filament_always_gripped: 1          # gear steppers pinch filament continuously
```

### What we could not verify

`printables.com` and the reseller pages for this MMU are on a network egress
blocklist in this environment, so the following are **left as placeholders you must
fill in from your own BOM / board pinout**, not real numbers:

- Exact MCU/board driving the MMU (commonly an SKR Pico, an EBB-style CAN toolboard,
  or the printer's spare motherboard headers — Happy Hare supports any of these as
  a Klipper `[mcu]`)
- Which stepper driver channels the 4 gate motors are wired to
- Whether your build includes a **filament encoder** (many low-part-count Type-B
  designs skip it and rely on gate sensors + collision/stallguard homing instead —
  this repo defaults to **no encoder**; uncomment the encoder block in
  `mmu_hardware_lowrider.cfg` if yours has one)
- Whether you have per-gate **pre-gate sensors** (recommended, and assumed present
  below) and/or a single shared **gate/exit sensor** at the point the 4 channels
  merge into the shared Bowden tube
- Bowden tube length from the MMU's merge point to the right toolhead's extruder
  entrance, and whether all 4 gates share the same length to that merge point
  (`variable_bowden_lengths: 0` assumes yes — flip to `1` if your gates have visibly
  different path lengths)

### Assumed sensor layout (edit if yours differs)

```
gate 0..3   pre-gate switch sensor   -> detects filament loaded into each gate
(merge)     shared gate sensor        -> optional, detects filament past the merge point
right toolhead   toolhead sensor      -> at/near the extruder entrance of the right toolhead,
                                          used for homing + runout + calibration reference
```

Fill in the actual pins for these in `config/happy-hare/mmu_hardware_lowrider.cfg`
— every placeholder is written as `<FILL_IN: description>` with a comment on what
to look for (motherboard silkscreen label, toolboard connector name, etc.).

## Toolboard note

Because the MMU-fed side is the *right* toolhead, its extruder stepper current,
rotation_distance, and any `sync_feedback_tension_pin`/`sync_feedback_compression_pin`
(if your MMU has a buffer/spring arm) belong on **that** toolhead's MCU, not the
MMU's own MCU. Keep the MMU's own MCU strictly for the 4 gate steppers + MMU-local
sensors; everything downstream of the merge point (toolhead sensor, extruder sync)
lives on the printer's main or toolboard MCU, per Happy Hare's own convention (see
the comment block at the top of `mmu.cfg` upstream: *"certain sensors (toolhead,
extruder, sync-feedback) are typically on the printer's main MCU and should be
configured separately"*).
