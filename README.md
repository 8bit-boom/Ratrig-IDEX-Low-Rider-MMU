# RatRig V-Core 4.1 IDEX + Low Rider MMU v2.0 (Happy Hare)

A working configuration scaffold for running a **RatRig V-Core 4.1 IDEX** printer on
**RatOS**, with a **Low Rider MMU v2.0** (the community, "for any Klipper printer"
design by GcodeGearhead) driven by **Happy Hare**, feeding filament into the
**right toolhead only**. The left toolhead stays a normal, independent,
single-material toolhead.

This repo does **not** replace any of the four upstream projects it wires together:

| Project | Role | Link |
|---|---|---|
| [RatOS](https://github.com/Rat-OS/RatOS) | Base Klipper/Moonraker/Mainsail image | github.com/Rat-OS/RatOS |
| [RatOS-configuration](https://github.com/Rat-OS/RatOS-configuration) | Generates the printer's core `RatOS.cfg` via the RatOS Configurator | github.com/Rat-OS/RatOS-configuration |
| [Happy Hare](https://github.com/moggieuk/Happy-Hare) | MMU control software (gate select, load/unload, tool state) | github.com/moggieuk/Happy-Hare |
| [Low Rider MMU v2.0](https://www.printables.com/model/1592898-low-rider-mmu-v20-for-any-klipper-printer) | Printable MMU hardware — Type-A, one shared gear stepper + one servo-driven CAM selector | printables.com/model/1592898 |

**What this repo adds:** the glue that a stock RatOS IDEX install and a stock Happy
Hare install don't provide out of the box — because both independently want to own
the G-code `T0`/`T1`/... namespace. See
[`docs/03-integration-architecture.md`](docs/03-integration-architecture.md) for why
that collides and how it's resolved here.

## Architecture at a glance

```
Slicer tool list
  T0      → RatOS physical carriage 0 (left toolhead / extruder), standalone, no MMU
  T1      → RatOS physical carriage 1, "as-is" -- reserved, not used in MMU projects
  T2..T5  → Happy Hare logical gates 0-3 on the Low Rider MMU
             → always physically printed by RatOS carriage 1 (right toolhead / extruder1)

Physical hardware
  Left toolhead  (carriage 0) -- direct/normal extruder, single filament, always available
  Right toolhead (carriage 1) -- fed exclusively by the Low Rider MMU

MMU mechanism (Type-A, per the official build guide -- NOT one stepper per gate)
  1x NEMA17 gear stepper, shared by all gates via an 80T gear/belt train
  1x MG996R servo rotating a CAM shaft ("ServoSelector") to engage one gate at a time
  1x BTT EBB42 v1.2 CAN toolboard, mounted on the MMU, drives all of the above
  Pre-gate switch per lane; no gate sensor, no encoder -- gate loads home against
  the RIGHT TOOLHEAD's own extruder-entry sensor instead
```

`T0`/`T1` are RatOS's own physical-carriage macros and are **never redefined** by
this repo — RatOS's internals (`PARK_TOOLHEAD`, `JOIN_SPOOLS`, IDEX copy/mirror,
etc.) hardcode lookups against those exact names, so the MMU's 4 gates get their
own names (`T2`-`T5`) instead. Full reasoning in
[`docs/03-integration-architecture.md`](docs/03-integration-architecture.md) — read
that one first, it explains a real collision this scheme avoids.

Gate count is fixed at **4** throughout this repo (`mmu_num_gates: 4`, using gates
0-3 of a possible 2-6 the Low Rider MMU supports). If your build has a different
gate count, see the note in
[`docs/03-integration-architecture.md`](docs/03-integration-architecture.md) — it's a
small, well-contained change.

## Repo layout

```
docs/
  01-hardware-overview.md        The Low Rider MMU's real mechanism/wiring, from the official build guide
  02-installation.md             RatOS -> RatOS Configurator -> Happy Hare install order
  03-integration-architecture.md The T2-T5 tool-numbering scheme and why it's needed (read this first)
  04-slicer-setup.md             OrcaSlicer setup: verified G-code hooks + the IDEX topology adaptation
  05-calibration.md              Calibration order: IDEX, then MMU (Type-A flow), then combined
config/
  printer-overrides.cfg          Goes into printer.cfg, below [include RatOS.cfg]
  happy-hare/
    mmu_pins_lowrider.cfg        [board_pins mmu] alias block for the MMU's EBB42, from the build guide
    mmu_parameters_lowrider.cfg  Happy Hare "MMX" vendor parameters for the Low Rider MMU
    mmu_hardware_lowrider.cfg    Gear stepper / selector servo / sensor pin config (Type-A, single gear+servo)
  macros/
    idex_mmu_integration.cfg     T2-T5 (MMU gates) macros + carriage-switch glue. T0/T1 stay RatOS stock.
  moonraker/
    update_manager_happy_hare.conf   Moonraker update_manager entry for Happy Hare
```

## Quick start

1. Flash RatOS, run the RatOS Configurator, select **V-Core 4.1 IDEX**, pick your
   boards/toolboards/hotends and generate `RatOS.cfg`. See
   [`docs/02-installation.md`](docs/02-installation.md).
2. Install Happy Hare (`git clone` + `./install.sh`), choosing MMU type **MMX**,
   MCU **BTT EBB 42 CANbus v1.2**, servo **MG996r**, and **4** gates. Same doc,
   step 3.
3. Copy the files from `config/happy-hare/` and `config/macros/` into
   `~/printer_data/config/` (destinations documented at the top of each file) and
   fill in the `<FILL_IN: ...>` placeholders — most of these are things the
   official build guide itself says must be measured per-machine (gear current,
   gate homing distances, servo angles, toolhead sensor pins/dimensions), not
   generic values. See [`docs/01-hardware-overview.md`](docs/01-hardware-overview.md).
4. Append `config/printer-overrides.cfg` to your `printer.cfg`, below
   `[include RatOS.cfg]` and below Happy Hare's own generated MMU includes.
5. Add the Happy Hare Moonraker entry from `config/moonraker/update_manager_happy_hare.conf`
   to `moonraker.conf`.
6. `RESTART`, then follow [`docs/05-calibration.md`](docs/05-calibration.md) in order.
7. Configure your slicer per [`docs/04-slicer-setup.md`](docs/04-slicer-setup.md).

## Provenance

The MMU-mechanism, wiring, and Happy Hare parameter values in `docs/01` and
`config/happy-hare/` are grounded in **GcodeGearhead's official Low Rider MMU
V2.0 Build Guide** (a 56-page PDF the user supplied directly, since
`printables.com` and the vendor's reseller pages are blocked by this
environment's network policy). Values the guide itself flags as build-specific
(servo angles, gate-homing distances, gear current, toolhead dimensions/pins,
cutter geometry) are left as `<FILL_IN: ...>` placeholders with an explanation of
what to measure — the guide is explicit that these vary machine to machine, so
this isn't a gap, it's what the source material itself says. Everything else —
the Happy Hare vendor/mechanism choice, the exact EBB42 pin-out, the calibration
sequence, the slicer G-code hooks, and the RatOS IDEX integration/tool-numbering
scheme — is grounded in that guide plus the actual upstream RatOS,
RatOS-configuration, RatOS-configurator, and Happy Hare source, not guessed.
