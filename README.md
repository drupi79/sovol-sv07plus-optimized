# Sovol SV07 Plus — Optimized Klipper Config & OrcaSlicer Profiles

> ⚠️ **WARNING — READ BEFORE USING**
>
> These configuration files are provided as-is and may or may not work correctly on your specific printer. Every SV07 Plus may differ in firmware version, hardware revision, probe calibration, and physical setup. **Always back up your original `printer.cfg` and OrcaSlicer profiles before applying any changes.** Test carefully after applying — start with manual movement commands before running a full print.
>
> The z_offset, PID values, and input shaper data in `printer.cfg` were calibrated on a specific machine and **will almost certainly need to be re-calibrated on yours.**

---

## Printer Overview

| Property | Value |
|----------|-------|
| Kinematics | Cartesian |
| Board | STM32F103, 28KiB bootloader, USART1 |
| Print Volume | 300 × 300 × 350 mm |
| Z Axis | Dual leadscrew (Z_TILT_ADJUST) |
| X/Y Homing | Sensorless (TMC2209 virtual endstop) |
| Probe | Inductive, x_offset: +27mm, y_offset: −20mm |
| Extruder | Direct drive, rotation_distance: 4.59 |
| Input Shaper | MZV @ 40.8Hz (X), MZV @ 32.0Hz (Y) |
| Pressure Advance | 0.02 |

---

## Files Included

| File | Description |
|------|-------------|
| `printer.cfg` | Optimized Klipper configuration |
| `printer_profile.json` | OrcaSlicer printer profile |
| `quality.json` | Quality process profile (slow, detailed) |
| `balanced.json` | Balanced process profile (default) |
| `speed.json` | Speed process profile (fast, functional) |
| `print_in_place.json` | Print-in-place process profile (functional prints, hinges, etc.) |

---

## What Was Changed (printer.cfg)

### Bug Fixes

| Issue | Fix |
|-------|-----|
| `max_accel_to_decel` (deprecated in Klipper) | Replaced with `minimum_cruise_ratio: 0.5` |
| `M205` macro that called `M105` and did nothing | Removed |
| `[z_tilt] points` left probe point at X=−8 — with x_offset=+27, the nozzle would need to reach X=−35 (below position_min=−12), causing a move-out-of-range error | Shifted to X=15 (nozzle at X=−12, probe at X=+15) |
| `[screws_tilt_adjust]` left screws at probe X=4 — same issue, nozzle would need X=−23 | Shifted to X=15 |
| Resonance tester scan range was 1–100Hz (too narrow for future re-calibration) | Widened to 5–133Hz |

### Macro Improvements

**PRINT_START** (replaces the near-empty original):
- Takes `EXTRUDER_TEMP=` and `BED_TEMP=` parameters from the slicer
- Starts bed heating, disables E motor (improves probe accuracy for sensorless homing)
- Homes all axes (`G28`)
- Runs `Z_TILT_ADJUST` to level dual Z leadscrews
- Re-homes Z after tilt correction
- Waits for bed temperature
- Runs `BED_MESH_CALIBRATE` (fresh probe mesh every print)
- Moves to purge position, heats extruder, runs purge line
- OrcaSlicer start gcode: `PRINT_START EXTRUDER_TEMP=[nozzle_temperature_initial_layer] BED_TEMP=[bed_temperature_initial_layer_single]`

**PRINT_END** (replaces the minimal original):
- Retracts filament, raises Z
- Parks toolhead at rear (X10, Y290)
- Turns off fan, heaters, and disables motors
- Saves PLR (power loss recovery) state

**START_PRINT / END_PRINT**: kept as thin aliases for backwards compatibility with any saved gcode that calls them.

**LOAD_FILAMENT / UNLOAD_FILAMENT**:
- Now accept a `TEMP=` parameter (default: 220°C)
- Heat nozzle to temperature before extruding/retracting
- Original macros extruded cold if run without pre-heating

**CANCEL_PRINT**: simplified to properly park and shut down before calling the base cancel command.

### Quality of Life Additions

- `[gcode_arcs] resolution: 0.1` — smoother arc interpolation (was missing entirely)
- `[exclude_object]` — enables per-object cancellation in Mainsail/Fluidd

### Preserved Unchanged

- All SAVE_CONFIG values (z_offset, PID tuning, input shaper frequencies, bed mesh)
- All pin assignments and stepper driver settings
- Sensorless homing thresholds (`driver_sgthrs: 95`)
- Probe offsets and safe_z_home position
- PLR (power loss recovery) include

---

## What Was Changed (OrcaSlicer Profiles)

### Printer Profile

- `machine_start_gcode` changed from inline heating + `START_PRINT` to:
  `PRINT_START EXTRUDER_TEMP=[nozzle_temperature_initial_layer] BED_TEMP=[bed_temperature_initial_layer_single]`
- `machine_end_gcode` changed to: `PRINT_END`

### Process Profiles

Four profiles are included, all with `bridge_flow: 0.95`, `elefant_foot_compensation: 0.1`, outer-first wall sequence, and organic tree support where applicable. Acceleration limits are derived from the input shaper data (MZV @ 32Hz on Y axis, the limiting axis).

| Profile | Default Accel | Outer Wall Accel | Outer Wall Speed | Walls |
|---------|--------------|------------------|------------------|-------|
| Quality | 1500 mm/s² | 1200 mm/s² | 40 mm/s | 3 |
| Balanced | 3000 mm/s² | 2500 mm/s² | — | 3 |
| Speed | 3400 mm/s² | 3000 mm/s² | — | 2 |
| Print-in-place | 800 mm/s² | 600 mm/s² | 30 mm/s | 4 |

The print-in-place profile additionally enables `precise_outer_wall`, seam slope smoothing, aligned seam, outer brim, and slow overhang speeds.

> **Note:** The process profiles use `"inherits": "0.20mm Standard @Sovol SV07 Plus"` — you must have the Sovol SV07 Plus profiles installed in OrcaSlicer for the inheritance to resolve correctly. Install them from OrcaSlicer's built-in printer marketplace if needed.

---

## How to Deploy

### Klipper (printer.cfg)

1. **Back up your existing config first:**
   ```
   cp ~/printer_data/config/printer.cfg ~/printer_data/config/printer.cfg.backup
   ```
2. Copy `printer.cfg` to your Pi:
   ```bash
   scp printer.cfg pi@<your-printer-ip>:~/printer_data/config/
   ```
3. Restart Klipper via Mainsail or:
   ```bash
   sudo systemctl restart klipper
   ```
4. After restart, run `SCREWS_TILT_ADJUST` from the Mainsail console to verify bed leveling works.
5. Run `Z_TILT_ADJUST` to verify dual-Z leveling works.
6. **Re-run PID calibration** if your heater hardware differs:
   ```
   PID_CALIBRATE HEATER=extruder TARGET=200
   PID_CALIBRATE HEATER=heater_bed TARGET=60
   ```
7. **Re-run probe z_offset calibration** — the saved z_offset is specific to the original machine.

### OrcaSlicer

1. Open OrcaSlicer → Printer settings → Import
2. Import `printer_profile.json`
3. Import each process profile (`quality.json`, `balanced.json`, `speed.json`, `print_in_place.json`)
4. Select **"Sovol SV07 Plus 0.4 nozzle Optimized"** as your printer
5. Start with the **Balanced** profile for your first print

---

## Disclaimer

These files were generated as an optimization pass over an existing printer backup. They have not been tested on a physical printer. Use at your own risk. The author takes no responsibility for any damage, failed prints, or hardware issues resulting from the use of these files.
