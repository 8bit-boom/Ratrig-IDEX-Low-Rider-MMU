# RatRig V-Core 4.1 IDEX + Low Rider MMU v2.0 (Happy Hare)

A working configuration scaffold for running a **RatRig V-Core 4.1 IDEX** printer on
**RatOS**, with a **Low Rider MMU v2.0** (the community, "for any Klipper printer"
design by GcodeGearHead) driven by **Happy Hare**, feeding filament into the
**right (T1) toolhead only**. The left toolhead (T0) stays a normal, independent,
single-material toolhead.

This repo does **not** replace any of the four upstream projects it wires together:

| Project | Role | Link |
|---|---|---|
| [RatOS](https://github.com/Rat-OS/RatOS) | Base Klipper/Moonraker/Mainsail image | github.com/Rat-OS/RatOS |
| [RatOS-configuration](https://github.com/Rat-OS/RatOS-configuration) | Generates the printer's core `RatOS.cfg` via the RatOS Configurator | github.com/Rat-OS/RatOS-configuration |
| [Happy Hare](https://github.com/moggieuk/Happy-Hare) | MMU control software (gate select, load/unload, tool state) | github.com/moggieuk/Happy-Hare |
| [Low Rider MMU v2.0](https://www.printables.com/model/1592898-low-rider-mmu-v20-for-any-klipper-printer) | Printable MMU hardware (4-gate, per-gate drive, no selector) | printables.com/model/1592898 |

**What this repo adds:** the glue that a stock RatOS IDEX install and a stock Happy
Hare install don't provide out of the box — because both independently want to own
the G-code `T0`/`T1`/... namespace. See
[`docs/03-integration-architecture.md`](docs/03-integration-architecture.md) for why
that collides and how it's resolved here.

## Architecture at a glance

```
Slicer tool list (6 "extruders")
  T0      → RatOS physical carriage 0 (left toolhead / extruder), standalone, no MMU
  T1      → RatOS physical carriage 1, "as-is" -- reserved, not used in MMU projects
  T2..T5  → Happy Hare logical gates 0-3 on the Low Rider MMU
             → always physically printed by RatOS carriage 1 (right toolhead / extruder1)

Physical hardware
  Left toolhead  (carriage 0) -- direct/normal extruder, single filament, always available
  Right toolhead (carriage 1) -- fed exclusively by the Low Rider MMU's 4 gates
```

`T0`/`T1` are RatOS's own physical-carriage macros and are **never redefined** by
this repo — RatOS's internals (`PARK_TOOLHEAD`, `JOIN_SPOOLS`, IDEX copy/mirror,
etc.) hardcode lookups against those exact names, so the MMU's 4 gates get their
own names (`T2`-`T5`) instead. Full reasoning in
[`docs/03-integration-architecture.md`](docs/03-integration-architecture.md) — read
that one first, it explains a real collision this scheme avoids.

Gate count is fixed at **4** throughout this repo (`mmu_num_gates: 4`). If your build
has a different gate count, see the note in
[`docs/03-integration-architecture.md`](docs/03-integration-architecture.md) — it's a
small, well-contained change.

## Repo layout

```
docs/
  01-hardware-overview.md        Wiring assumptions for the Low Rider MMU v2.0 + what to verify against your BOM
  02-installation.md             RatOS -> RatOS Configurator -> Happy Hare install order
  03-integration-architecture.md The T2-T5 tool-numbering scheme and why it's needed (read this first)
  04-slicer-setup.md             OrcaSlicer/PrusaSlicer/SuperSlicer multi-material profile setup
  05-calibration.md              Calibration order: IDEX, then MMU, then combined
config/
  printer-overrides.cfg          Goes into printer.cfg, below [include RatOS.cfg]
  happy-hare/
    mmu_parameters_lowrider.cfg  Happy Hare "Other" vendor parameters for the Low Rider MMU
    mmu_hardware_lowrider.cfg    Pin-out template for the Low Rider MMU's 4 gate steppers/sensors
  macros/
    idex_mmu_integration.cfg     T2-T5 (MMU gates) macros + carriage-switch glue. T0/T1 stay RatOS stock.
  moonraker/
    update_manager_happy_hare.conf   Moonraker update_manager entry for Happy Hare
```

## Quick start

1. Flash RatOS, run the RatOS Configurator, select **V-Core 4.1 IDEX**, pick your
   boards/toolboards/hotends and generate `RatOS.cfg`. See
   [`docs/02-installation.md`](docs/02-installation.md).
2. Install Happy Hare (`git clone` + `./install.sh`), choosing MMU vendor **Other**,
   selector type **VirtualSelector**, **4** gates, and toolhead extruder **extruder1**
   (the right/T1 toolhead). Same doc, step 2.
3. Copy the files from `config/` into `~/printer_data/config/` (paths documented at
   the top of each file) and fill in the pin placeholders against your actual
   toolboard/MCU pinout — see [`docs/01-hardware-overview.md`](docs/01-hardware-overview.md).
4. Append `config/printer-overrides.cfg` to your `printer.cfg`, below
   `[include RatOS.cfg]` and below Happy Hare's own generated MMU includes.
5. Add the Happy Hare Moonraker entry from `config/moonraker/update_manager_happy_hare.conf`
   to `moonraker.conf`.
6. `RESTART`, then follow [`docs/05-calibration.md`](docs/05-calibration.md) in order.
7. Configure your slicer per [`docs/04-slicer-setup.md`](docs/04-slicer-setup.md).

## Important caveat

Printables and the Low Rider MMU vendor's own product pages were not reachable while
building this (network policy blocks those domains from this environment), so the
**exact pin-out, gate count default, and BOM** of your specific Low Rider MMU v2.0
build could not be pulled automatically. Every pin in `config/happy-hare/` is a named
placeholder (`<FILL_IN: ...>`) with a comment explaining what to measure/look up on
your own board — this mirrors how Happy Hare's own upstream templates work (they are
never distributed with real pins filled in either). Everything else — the Happy Hare
parameter choices, the vendor/selector type, the IDEX/MMU tool-numbering scheme, the
macro logic, and the include order — is grounded in the actual upstream RatOS,
RatOS-configuration, RatOS-configurator, and Happy Hare source, not guessed.
