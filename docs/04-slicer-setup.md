# Slicer setup (OrcaSlicer)

GcodeGearhead's build guide documents OrcaSlicer setup for a **single-toolhead**
printer (Happy Hare's normal case: one physical extruder, "Single Extruder Multi
Material" mode, `Extruders: 1`). The G-code hooks below are reproduced from those
screenshots and are correct regardless of your printer's toolhead count — use
them as-is. The **printer/extruder topology** section further down is where a
single-toolhead recipe has to be adapted for IDEX, and that adaptation is *not*
something either the build guide or the Happy Hare wiki documents (both assume
one nozzle) — treat that part as a starting point to verify against your own
OrcaSlicer version, not copied gospel.

## Verified: Happy Hare G-code hooks (from the build guide)

Machine start G-code — keep your existing bed-temp line, then add the Happy Hare
block (this is what actually starts the MMU for the print, mapping the slicer's
tool/color/purge-volume data into Happy Hare):

```
M140 S[bed_temperature_initial_layer_single] ;SET BED TEMP
;***********************************
MMU_START_SETUP INITIAL_TOOL={initial_tool} TOTAL_TOOLCHANGES=!total_toolchanges! REFERENCED_TOOLS=!referenced_tools! TOOL_COLORS=!colors! TOOL_TEMPS=!temperatures! TOOL_MATERIALS=!materials! FILAMENT_NAMES=!filament_names! PURGE_VOLUMES=!purge_volumes!
MMU_START_CHECK
;***********************************
```

Layer change G-code:

```
_MMU_UPDATE_HEIGHT
; If you want enhanced pausing feature with Happy Hare client macros also add this
SET_PRINT_STATS_INFO CURRENT_LAYER={layer_num} ; For pause at layer functionality and better print stats
```

Machine end G-code:

```
MMU_END
END_PRINT
```

Pause G-code:

```
PAUSE
```

Change filament G-code: leave empty — Happy Hare's `Tn` macros handle the entire
toolchange, nothing extra is needed here.

## Verified: right toolhead (MMU-fed) extruder settings

These apply to whichever slicer "extruder" ends up representing your MMU-fed
right toolhead (see topology note below). From the build guide's own working
profile:

- Retraction length: 0.5mm, speed/deretraction 30mm/s, travel threshold 1mm
- Retract on layer change + retract on top layer: on
- Wipe while retracting: on, wipe distance 2mm
- Z-hop: Normal, 0.4mm, on all surfaces
- Single extruder multi-material parameters: cooling tube position/length 0,
  filament parking position 0, extra loading distance 0, high extruder current
  on filament swap: off
- Advanced: filament load time 20s, filament unload time 20s, tool change time 0s
- Wipe tower: purge in prime tower on, filament ramming off (Happy Hare purges
  its own way — let the slicer's ramming stay off)

Reference: `https://github.com/moggieuk/Happy-Hare/wiki/Slicer-Setup`.

## Needs verification on your machine: printer/extruder topology

The build guide's own OrcaSlicer profile is `Extruders: 1` with **Single
Extruder Multi Material** checked — correct for their single-toolhead printer,
but that checkbox assumes the whole machine has exactly one physical extruder,
which isn't true for an IDEX printer with an independent standalone toolhead
alongside the MMU-fed one.

The concept that needs to survive the adaptation, regardless of exact UI
mechanics in your OrcaSlicer version:

- **2 physical extruders** in the printer profile: extruder 1 = left toolhead
  (standalone), extruder 2 = right toolhead (MMU-fed).
- **`Tn` in the G-code is a per-*filament* slot number, not a per-extruder
  number** — OrcaSlicer lets multiple filament profiles in a project share the
  same physical extruder, and each gets consecutive `T`-numbers regardless. That
  means, in principle, filament slot 0 (`T0`) → extruder 1 (left, standalone),
  and filament slots 2-5 (`T2`-`T5`) → extruder 2 (right, MMU gates 0-3), with
  slot 1 (`T1`) deliberately left unassigned/unused in every project (see
  `docs/03-integration-architecture.md` for why `T1` must stay reserved).
- Whether your OrcaSlicer version exposes "multiple filaments on one extruder,
  with the multi-material extruder flagged for MMU/AMS-style handling" through
  the SEMM checkbox scoped per-extruder, through the physical printer wizard, or
  through a different mechanism has changed across releases — check current
  OrcaSlicer docs/release notes rather than assuming the single-toolhead
  screenshots above translate literally. Build a minimal test project first (see
  below) and confirm the exported `.gcode` actually emits `T2`-`T5` (never `T0`
  from the MMU side, never `T1` at all) before trusting a real print to it.

## First test print

Before a full multi-material job, print something trivial that only exercises the
new numbering:

1. A single-color file assigned entirely to extruder 1 / filament slot 0 (`T0`) —
   confirms the standalone left toolhead is unaffected by any of this.
2. A single-color file assigned entirely to filament slot 2 (`T2`) — confirms
   `T2` correctly swaps to the right carriage and loads MMU gate 0.
3. A short multi-color file using filament slots 2-4 (`T2`-`T4`) only (no `T0`)
   — confirms in-place gate switching without carriage swaps.
4. Only once 1-3 pass, try a file mixing `T0` with `T2`-`T5`.

Open each exported `.gcode` and grep for the `Tn` lines before sending it to the
printer, to confirm the mapping came out the way you intended.
