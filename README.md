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
2. **Get bed mesh** reads the mesh currently active in the printer and fills everything in. It sends no motion or heating command. The top banner then shows the date, time and number of samples of the mesh on the sheet (1 sample for a single read, the number of runs for an averaged test).
3. **Averaged mesh test** runs the number of probings set in *Runs* (10 by default, at least 2) and uses their average for everything:
   - With **Real test unchecked** (the default) it is a mock: it reads the current mesh once and adds Gaussian noise (0.02 mm) to simulate the runs, one second per run. Nothing is heated or moved, so it is safe while the printer is busy.
   - **Console messages**: each step (homing, heating, soak, `run 3 of 10`, shutdown, result) is also printed in the Mainsail console with the `M7001` macro of [`ORCA_SLICER_GCODE.md`](ORCA_SLICER_GCODE.md), the mock included. The *CONSOLE_MESSAGES* parameter (Yes/No) turns it off. The macro must be in the printer configuration (and the printer rebooted); otherwise the messages are silently lost, the test is not affected.
   - With **Real test checked** it asks for confirmation, then drives the printer: `G28`; `MOVE_HEAT_POS` (park away from the bed before anything is heated, so a drooling nozzle cannot drip on it); bed to 55 °C and nozzle to 170 °C; `G9112` (nozzle wipe); `MOVE_HEAT_POS` again (Z to 15 mm, then X457, so the nozzle does not wait on the wipe pad); wait for the bed and nozzle down to 140 °C; 3 minute heat soak; `M7000 BED_MESH_CALIBRATE PROFILE=shimtest` once per run, reading the mesh after each (the `M7000` macro of [`ORCA_SLICER_GCODE.md`](ORCA_SLICER_GCODE.md) must be installed, the test checks it and stops before heating if it is missing; the test also stops if a mesh read is identical to the previous one, which would mean the probing did not run). **Why `M7000`:** the Rinkhals Moonraker (`kobra.py`, `patch_bed_mesh`) replaces any `gcode/script` that starts with `BED_MESH_CALIBRATE` by a full leveling sequence (`MOVE_HEAT_POS`, bed 60 °C and nozzle 170 °C from `[leviQ3]`, wipe, nozzle 140 °C, a bare `BED_MESH_CALIBRATE` without `PROFILE`, `TURN_OFF_HEATERS`, `M106 S0`, `SAVE_CONFIG`). A bare command would therefore heat and wipe again before every run, turn the heaters off after each probing, and overwrite and save the `default` mesh; wrapped in `M7000` the command reaches the firmware unchanged; then `TURN_OFF_HEATERS`, `M84`, `G9113`, and `BED_MESH_PROFILE LOAD=default`. The `default` profile is never overwritten and nothing is saved to disk. Excel is no longer frozen while a printer command runs (the requests are asynchronous and the macro yields to Excel every 50 ms), the status bar shows progress, and the **Stop** button next to *Averaged mesh test* ends the test at the next checkpoint (between two commands, or within a second during the heat soak; a command already running on the printer is not interrupted) and shuts the printer down. Do not edit the sheet during a test: edits of the shim table are ignored until it ends. If a step fails, the shutdown is still attempted.
4. The **+** and **−** buttons right of the strip list add or remove one strip (relative to the strips listed), keep the current mesh and shims, and recompute the tape drawing and the third heat map. The number is also written in *Max strips* on the Parameters sheet. **+** adds the next best strip even when it gains less than *TAPE_STOP*, and says so when no strip reduces the spread any more. Editing a shim afterwards recomputes with the normal rule again.
5. Read the shim table (back/front × left/right), the strip table and drawing, and compare the three heat maps.

### Assumptions to check

- A higher mesh value means the bed is higher at that point. If your firmware uses the opposite sign, the shims belong on the opposite corners.
- Mesh row 0 is the front and column 0 is the left side, seen from the front.
- The flexible plate follows the tape smoothly (edges softened over about 5 mm); a stiff real plate may bridge narrow strips, so the tape simulation is optimistic.
- Bed size, screw positions, tape thickness and width, temperatures and soak time are constants at the top of the macro (Alt+F11).

## Status

All 28 chapters of the guide are written: ACE hub, print macros, bed mesh, calibration, core G-code, exclude object, extruder, factory commands, fans, filament sensor, heaters, homing, input shaper, LeviQ3, low-level motion, macros, misc, moves and modes, position/offsets/saved state, OTA, pause/resume/cancel, print job, print statistics, probe, queries, TMC drivers, timelapse, toolhead limits. Findings marked **(to verify)** are still open.

## What is not in this repository

Only the Markdown documents and `Bed Leveling Planner.xlsm` are tracked. The firmware files, the analysis scripts and data, the clone of the public source, and local test work (such as the sharp corners tuning files) are deliberately left out.
