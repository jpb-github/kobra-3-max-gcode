# Anycubic Kobra 3 Max — G-code, macros and Orca Slicer setup

Documentation for the Anycubic Kobra 3 Max (firmware **2.5.2.8**, "Klipper-Go"):

- a user-friendly guide to its G-code commands and macros, written from an analysis of the firmware binary (`gklib`); the public Klipper-Go source on GitHub is out of date, so it was used only as a hint, never as the source of truth;
- the Orca Slicer start/end G-code built from that guide;
- the customized Orca Slicer profiles compared with the stock ones.

> **Nothing in these documents has been test-run on a printer unless stated.** Statements that rest on inference are marked **(to verify)**. Do not send G-code to your printer on the strength of this documentation alone.

## Contents

| File | What it is |
|---|---|
| [`K3M_gcode_user_guide.md`](K3M_gcode_user_guide.md) | **The G-code guide.** One chapter per command family (28 chapters): what each command does step by step, every parameter (meaning, values, default), examples, cautions. It also documents how the firmware treats print files (which lines are run, what is silently skipped) and the velocity and acceleration limits. |
| [`K3M_2.5.2.8_gcode_commands.md`](K3M_2.5.2.8_gcode_commands.md) | Technical reference: the full table of the 175 registered commands, how the list was extracted from the firmware, and the key findings. |
| [`ORCA_SLICER_GCODE.md`](ORCA_SLICER_GCODE.md) | **Orca Slicer G-code for this printer**: machine start, machine end, before-layer-change and layer-change blocks (wipe, LeviQ3 Z offset, adaptive bed mesh, pressure advance from the filament profile, console messages), the two one-time macros (`M7000`, `M7001`) to add to the printer configuration, and the reasoning behind every step. |
| [`ORCA_PROFILE_CHANGES.md`](ORCA_PROFILE_CHANGES.md) | **The tuned Orca profiles**: every setting that differs from the stock profiles in the printer profile (`Kobra 3 Max - Network`), the process profile (`0.20mm Kobra 3 Max 0.4 - Multiboard`) and the filament profile (`Grossiste 3D PLA Matte @System`), with a short note on each, the speeds and accelerations chosen for this printer, and a reminder to calibrate flow and pressure advance for each filament. The start/end G-code is left to `ORCA_SLICER_GCODE.md`. |
| [`Bed Leveling Planner.xlsm`](Bed%20Leveling%20Planner.xlsm) | **An Excel workbook (with macros) to level the bed.** It reads the bed mesh from the printer, shows it as a heat map, and works out the screw shims and Kapton tape strips that would flatten it. See [Bed Leveling Planner](#bed-leveling-planner) below. |

## Bed Leveling Planner

`Bed Leveling Planner.xlsm` is a single-sheet Excel workbook for a bed that is held by four fixed screws (shims, no leveling screws) and carries a flexible steel plate. It needs Windows Excel with macros enabled and a printer reachable at `192.168.1.46` (Moonraker on port 7125; change `PRINTER_URL` at the top of the macro if yours differs).

> **The macros have not been run end to end.** The workbook was built by script and the sheet layout was checked, but the buttons were never clicked in Excel and the real test never ran on a printer. Try the mock test first.

### What it does

- **Reads the mesh**: the 9 × 9 bed mesh of the printer, displayed with the front on the last row and the left side in the first column, rounded to two decimals, as a heat map (blue = lowest, red = highest). The numbers around the matrix are the tilt of each side (half the height difference between its two corners).
- **Screw shims**: suggests four shim thicknesses (0.05 mm steps) for the screws, assumed 50 mm from the left/right sides and 70 mm from the front/back of a 420 × 420 mm bed. The bed is treated as rigid, so shims can only tilt it, not remove its warp. The second heat map simulates the result.
- **Kapton tape**: under the flexible plate, places up to 3 strips (set in *Max strips*, up to 8) of 20 mm wide tape, 0.065 mm per layer, up to 4 layers each, where they reduce the spread most. A drawing shows the strips on the bed with the coordinates of their ends (mm from the front-left corner), and a table lists each strip (ends, length, angle, layers). The third heat map simulates shims plus tape. The three heat maps share one colour scale, and a summary gives the peak-to-peak and standard deviation of each stage.
- **Averaged test**: runs several probings and averages them, and reports the variance between runs, its standard deviation and a data-quality rating (Excellent ≤ 0.01 mm, Good ≤ 0.02, Fair ≤ 0.04, Poor above).

### How to use it

1. Open the file and click **Enable Content** (the macros are needed for the buttons).
2. **Get bed mesh** reads the mesh currently active in the printer and fills everything in. It sends no motion or heating command.
3. **Averaged mesh test** runs the number of probings set in *Runs* (10 by default, at least 2) and uses their average for everything:
   - With **Real test unchecked** (the default) it is a mock: it reads the current mesh once and adds Gaussian noise (0.02 mm) to simulate the runs. Nothing is heated or moved, so it is safe while the printer is busy.
   - With **Real test checked** it asks for confirmation, then drives the printer: `G28`; bed to 55 °C and nozzle to 170 °C; `G9112` (nozzle wipe); wait for the bed and nozzle down to 140 °C; 3 minute heat soak; `BED_MESH_CALIBRATE PROFILE=shimtest` once per run, reading the mesh after each; then `TURN_OFF_HEATERS`, `M84`, `G9113`, and `BED_MESH_PROFILE LOAD=default`. The `default` profile is never overwritten and nothing is saved to disk. Excel is frozen while a printer command runs; the status bar shows progress. If a step fails, the shutdown is still attempted.
4. Read the shim table (back/front × left/right), the strip table and drawing, and compare the three heat maps.

### Assumptions to check

- A higher mesh value means the bed is higher at that point. If your firmware uses the opposite sign, the shims belong on the opposite corners.
- Mesh row 0 is the front and column 0 is the left side, seen from the front.
- The flexible plate follows the tape smoothly (edges softened over about 5 mm); a stiff real plate may bridge narrow strips, so the tape simulation is optimistic.
- Bed size, screw positions, tape thickness and width, temperatures and soak time are constants at the top of the macro (Alt+F11).

## Status

All 28 chapters of the guide are written: ACE hub, print macros, bed mesh, calibration, core G-code, exclude object, extruder, factory commands, fans, filament sensor, heaters, homing, input shaper, LeviQ3, low-level motion, macros, misc, moves and modes, position/offsets/saved state, OTA, pause/resume/cancel, print job, print statistics, probe, queries, TMC drivers, timelapse, toolhead limits. Findings marked **(to verify)** are still open.

## What is not in this repository

Only the Markdown documents and `Bed Leveling Planner.xlsm` are tracked. The firmware files, the analysis scripts and data, the clone of the public source, and local test work (such as the sharp corners tuning files) are deliberately left out.
