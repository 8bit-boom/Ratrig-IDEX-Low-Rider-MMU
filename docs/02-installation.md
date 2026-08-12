# Installation order

Install in this order — later steps assume earlier ones are done.

## 1. RatOS

Flash the latest RatOS image and boot the printer's Pi/CM4. Follow
`os.ratrig.com` first-boot instructions to reach Mainsail and the RatOS
Configurator (bundled, served by the `ratos-configurator` Moonraker service).

## 2. RatOS Configurator: generate `RatOS.cfg`

In the Configurator web UI:

1. Printer: **RatRig V-Core 4.1 IDEX**, pick your build volume (300/400/500).
2. Control board + toolboards: pick what you actually have on **both** toolheads
   (defaults are BTT Octopus 1.1 + 2x BTT EBB42-12 CAN toolboards, but choose
   whatever your build uses).
3. Extruders/hotends/probe: set per toolhead as usual — this is unrelated to the
   MMU and works exactly like any stock IDEX build.
4. Finish the wizard. This writes `~/printer_data/config/RatOS.cfg` (generated,
   don't hand-edit it) and a starter `~/printer_data/config/printer.cfg` that
   begins with `[include RatOS.cfg]`.

At this point you should have a fully working, MMU-free dual-toolhead IDEX printer.
**Get that working and calibrated first** (home, PID tune, input shaping via
`GENERATE_SHAPER_GRAPHS` — `SHAPER_CALIBRATE` is disabled on IDEX, per RatOS's own
`idex_is.cfg` — and dial in `T0`/`T1` toolchange offsets) before adding the MMU.
Debugging IDEX and MMU problems at the same time is much harder than debugging them
one at a time.

## 3. Happy Hare

```bash
cd ~
git clone https://github.com/moggieuk/Happy-Hare.git
cd Happy-Hare
./install.sh
```

The installer is interactive. Answer:

- **MCU**: whichever board actually drives your MMU's 4 gate steppers (see
  `docs/01-hardware-overview.md` — this is *not* necessarily the printer's main
  board).
- **Vendor**: `Other`
- **Num gates**: `4`
- **Selector type**: `VirtualSelector` (no physical selector — the Low Rider MMU
  uses one gear stepper per gate instead)
- **Toolhead extruder**: point it at `extruder1` (the RatOS **right** toolhead).
  If the installer doesn't ask this directly, it's set explicitly in
  `config/happy-hare/mmu_parameters_lowrider.cfg` (`extruder: extruder1`) — make
  sure that override file is included **after** Happy Hare's own generated
  `mmu_parameters.cfg` so it wins.

This generates `~/printer_data/config/mmu/` with `mmu.cfg`, `mmu_hardware.cfg`,
`mmu_parameters.cfg`, `mmu_macro_vars.cfg`, etc., and adds
`[include mmu/base/*.cfg]`-style includes to your `printer.cfg`.

**Required manual step:** open the generated `~/printer_data/config/mmu/mmu_macro_vars.cfg`
and delete (or comment out) the auto-generated `[gcode_macro T0]` through
`[gcode_macro T3]` blocks (each just calls `MMU_CHANGE_TOOL TOOL=n`). These collide
with RatOS's own `T0`/`T1` physical-carriage macros — see
`docs/03-integration-architecture.md` for why that's unsafe to leave in place.
`config/macros/idex_mmu_integration.cfg` (step 4 below) supplies the replacements,
named `T2`-`T5`. Repeat this check after any future Happy Hare update that touches
`mmu_macro_vars.cfg`.

## 4. Drop in this repo's overrides

Copy this repo's files onto the generated tree:

```
config/happy-hare/mmu_parameters_lowrider.cfg  -> ~/printer_data/config/mmu/mmu_parameters_lowrider.cfg
config/happy-hare/mmu_hardware_lowrider.cfg    -> ~/printer_data/config/mmu/mmu_hardware_lowrider.cfg
config/macros/idex_mmu_integration.cfg         -> ~/printer_data/config/RatOS/idex_mmu_integration.cfg
```

(Any destination path is fine as long as the `[include ...]` lines in
`config/printer-overrides.cfg` are updated to match.)

Fill in every `<FILL_IN: ...>` placeholder in `mmu_hardware_lowrider.cfg` against
your actual wiring (see `docs/01-hardware-overview.md`).

## 5. Wire up printer.cfg and moonraker.conf

- Append the contents of `config/printer-overrides.cfg` to `printer.cfg`, **after**
  `[include RatOS.cfg]` and **after** Happy Hare's own generated MMU includes (so
  the parameter/hardware overrides win — Klipper's config system lets a later
  section of the same name override an earlier one, which is exactly the mechanism
  both RatOS and Happy Hare rely on for user overrides). This also declares our
  new `T2`-`T5` macros, which only need to come after `mmu.cfg` so
  `MMU_CHANGE_TOOL` already exists.
- Append `config/moonraker/update_manager_happy_hare.conf` to `moonraker.conf`
  (Happy Hare's installer may already have added this — check for a duplicate
  `[update_manager Happy-Hare]` section before appending).

## 6. Restart and verify

`RESTART` Klipper (not a full reboot). Check the console for errors. Run
`HELLO_RATOS` and `MMU_STATUS` — both should succeed with no config errors before
you proceed to `docs/05-calibration.md`.
