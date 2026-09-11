# Using a 0.2mm nozzle on the V-Core 4.1 IDEX

RatRig's own published slicer profiles and BOM only cover **0.4 / 0.5 / 0.6 /
0.8mm** nozzles. This doc covers what "officially only 0.4/0.6/0.8mm" actually
means, whether a 0.2mm nozzle physically fits your hotend, and how to configure
RatOS/Klipper and your slicer if you go ahead. Research for this doc hit the same
network restriction as the rest of this repo (most vendor/community sites were
unreachable), so **read the confidence level on each claim** — this is written to
be honest about what could and couldn't be verified, not to paper over gaps.

## Bottom line

- **"Officially supported" is a RatRig marketing/BOM statement, not a Klipper or
  RatOS firmware limit.** RatOS's own configurator accepts any nozzle diameter
  from 0.2mm to 1.8mm as a plain number field — see "RatOS/Klipper side" below.
- **Whether a 0.2mm nozzle physically fits depends on which Rapido variant you
  have** — HF vs UHF matters a lot here (next section).
- **No one has documented actually running a 0.2mm nozzle on a Rapido-family
  hotend**, good or bad. Treat this as genuinely unvalidated territory, not a
  known-working config with obscure settings.

## Does a 0.2mm nozzle exist and fit your hotend?

RatRig's V-Core 4.1 IDEX BOM uses Phaetus Rapido-family hotends, in two relevant
flavors:

- **Rapido HF / Rapido Plus (short, V6-length nozzle).** *(High confidence,
  corroborated across multiple independent reseller listings.)* These use a
  standard **V6-pattern, M6×1 thread** — the same interface as an E3D V6, not a
  proprietary thread. A V6-compatible 0.2mm nozzle from any brand should
  physically screw in. Phaetus's own general-purpose "Hardened Steel Nozzle"
  V6-pattern line reportedly spans 0.2mm-1.2mm per their product listing
  ([phaetus.com/en-us/products/hardened-steel-nozzle](https://www.phaetus.com/en-us/products/hardened-steel-nozzle))
  — *moderate confidence*, this was read from a search-engine snippet, not a
  direct page fetch, so verify the current catalog yourself before ordering.
  E3D's own official V6 range separately starts at 0.25mm; various third
  parties (Trianglelab, Micro Swiss, generic sellers) sell 0.2mm V6/MK8-pattern
  nozzles.
- **Rapido UHF / Rapido Plus UHF (Volcano-length nozzle).** *(High confidence.)*
  These ship with a longer, Volcano-length melt zone and nozzle, needing an
  adapter/different z-offset to run a short V6-length nozzle instead. E3D's own
  official Volcano nozzle line does not go below 0.6mm — there is no official
  small-diameter Volcano nozzle. Third-party "Volcano-style 0.2mm" nozzles exist
  (e.g. Vision Miner) but are unverified generic parts, not Phaetus/E3D
  products.

**Practical takeaway:** if your toolhead runs Rapido **HF**, a 0.2mm nozzle is a
plausible drop-in physical fit (standard V6 thread). If it runs Rapido **UHF**
(check `flowType` in your hotend selection — RatOS's `rapido-uhf.cfg` /
`rapido-plus-uhf.cfg` vs `rapido.cfg`), you're looking at either an unverified
generic Volcano-style nozzle or swapping to a V6-length setup, which is a bigger
change than just a nozzle swap.

## Why isn't 0.2mm on RatRig's official list?

No RatRig statement explaining the reasoning (flow mismatch, clog risk, etc.)
could be found — this was searched for specifically and came up empty across
RatRig's wiki, GitHub, and their upstreamed OrcaSlicer profile PRs. What *is*
confirmed directly (via GitHub) is that RatRig's own contributed OrcaSlicer
machine/process profiles for the V-Core 4 IDEX cover exactly 0.4/0.5/0.6/0.8mm,
and their Rapido hotends ship stock with 0.4mm and 0.6mm nozzles only. The
simplest explanation the evidence supports is **"RatRig profiles/validates what
they stock and sell,"** not a discovered technical incompatibility — but this is
an inference, not a sourced RatRig statement, so hold it loosely.

## RatOS/Klipper side: this is genuinely supported

Checked directly against the RatOS Configurator's own source
(`src/zods/hardware.tsx`): nozzle diameter is a **plain validated number field,
`min(0.2).max(1.8)`** — not a locked dropdown of "approved" sizes. Changing it
only changes one line, `nozzle_diameter:`, in the generated extruder config
(`src/server/helpers/config-generation/toolhead.ts`). There is no separate
firmware gate, no hotend-specific size whitelist, and no automatic flow-limit
scaling tied to nozzle diameter (`max_extrude_cross_section` isn't touched by
this field). So: **as far as RatOS/Klipper is concerned, 0.2mm is a normal,
supported input** — the limitation you're running into is entirely about
physical nozzle availability/fit for your specific hotend variant, covered
above, not software.

## Where to put it, given this repo's MMU setup

This repo's Happy Hare integration (`docs/01`-`05`) feeds the Low Rider MMU into
the **right toolhead**. If you're adding a 0.2mm nozzle, think about which side
makes sense before committing:

- **Right toolhead (MMU-fed) at 0.2mm:** every multi-material purge/tower move
  now happens through a nozzle with dramatically lower max volumetric flow (see
  below) — purge volumes that already cost time on a 0.4mm nozzle get
  considerably slower, and Happy Hare's toolchange/purge parameters
  (`mmu_macro_vars.cfg` purge speed, `extruder_purge_current`, etc.) tuned for a
  0.4mm nozzle will need lower flow targets too.
- **Left toolhead (standalone) at 0.2mm:** keeps the MMU path on a
  well-understood, higher-flow-tolerant 0.4/0.6mm nozzle, and gives you a
  dedicated fine-detail toolhead for single-material work. This is the lower-risk
  place to experiment first, since it doesn't add fine-nozzle flow constraints on
  top of an already-complex MMU toolchange path.

Either way, RatOS configures nozzle diameter **per toolhead independently** — the
IDEX profile already has two separate `[extruder]`/`[extruder1]` sections, so
running different nozzle sizes on each side is normal, not a special case.

## RatOS/Klipper configuration changes

1. In the RatOS Configurator, set the nozzle diameter for the chosen toolhead to
   `0.2`, type `Regular` (not `CHT` — CHT's flow-diffusing profile isn't made in
   sizes this small; use `Regular`).
2. **Re-run PID tuning is not required** (nozzle size doesn't change heater
   behavior), but **pressure advance must be recalibrated** for that toolhead —
   RatOS's stock Rapido default (`pressure_advance: 0.03`, in
   `hotends/rapido.cfg`/`rapido-plus-uhf.cfg`) was tuned against a 0.4mm nozzle's
   flow characteristics and will be wrong for 0.2mm. Redo Klipper's standard
   `TUNING_TOWER`/pressure advance calibration after the swap.
3. **Redo Z-offset / first-layer calibration.** A different physical nozzle tip
   changes your probe-to-nozzle offset even if the mounting thread is identical
   — don't reuse your 0.4mm nozzle's Z-offset.
4. Rotation distance / e-steps do **not** need to change (that's a filament-drive
   mechanical property, not nozzle-dependent).
5. If you're running Klipper's `[firmware_retraction]` (RatOS default,
   `retract_length: 0.5`), expect to retune this smaller — see retraction
   guidance below.

## Slicer settings

General FDM tuning consensus for 0.2mm nozzles (not RatRig-specific — treat as a
starting point, then tune on your machine). Confidence is mixed since most
vendor knowledge-base pages were unreachable while researching this; numbers
below were cross-checked across multiple independent sources where possible.

| Setting | 0.2mm nozzle | 0.4mm nozzle (for comparison) | Confidence |
|---|---|---|---|
| Layer height | ~0.05-0.15mm (25-75% of nozzle diameter; ~0.16mm is a commonly cited practical max) | 0.1-0.3mm | Moderate-high |
| Line width | ~0.20-0.22mm (100-110% of diameter) | 0.42-0.45mm | Moderate-high |
| Print speed | Flow-limited more than speed-limited — commonly cited working range roughly 40-90mm/s | 150-300mm/s on modern hardware | Moderate |
| Max volumetric flow | Dramatically lower — community/vendor figures cited around **~2mm³/s** (PLA, Bambu's own profile data) up to a reported failure point near **~5.25mm³/s** in independent testing | Commonly 11-15mm³/s stock, higher on upgraded hotends | Moderate — verify with your own flow test (Ellis' Print Tuning Guide method) before trusting a number |
| Retraction | Shorter distance than 0.4mm is typical (less molten volume near the tip), but the right number is extruder/Bowden-vs-direct-drive dependent — treat any fixed number as a starting point | RatOS default 0.5mm (direct drive, firmware retraction) | Low — retune empirically |
| Part cooling | Aggressive, early fan (near 100% after the first couple of layers) — thin layers have very little thermal mass and re-melt/warp easily without it | Moderate/ramped fan curve | Moderate |
| Nozzle temperature | Usually similar to, or a few degrees below, your 0.4mm setting for the same material — flow/speed is the real lever, not temperature | Material-standard | Moderate |

**Do your own max-volumetric-flow test** (Ellis' Print Tuning Guide method —
print a flow tower and watch for under-extrusion) before committing to a speed
profile; the table above is a starting point, not a substitute for calibration
on your specific hotend/filament combination.

## Risk factors to go in aware of

- Rapido-family hotends already have a documented community history of
  heat-creep/clogging complaints **at stock nozzle sizes** (RatRig Discord
  threads on Rapido 2 UHF clogging, heatbreak issues). This isn't evidence that
  a 0.2mm nozzle specifically makes it worse, but it's relevant background risk
  — a 0.2mm orifice gives you less margin for a partial clog to still pass
  filament, so problems that were a minor inconvenience at 0.4mm can become a
  full jam at 0.2mm.
- No one has published a report of running 0.2mm on a Rapido-family hotend that
  this research could find, positive or negative. You would be establishing
  this configuration for the RatRig community, not following a known-working
  path.
- If you go the "generic third-party Volcano-style 0.2mm nozzle on a UHF
  hotend" route, quality/tolerance consistency on ultra-fine generic nozzles is
  a known variable-quality space — inspect the orifice before use if possible.
