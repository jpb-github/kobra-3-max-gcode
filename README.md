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

## Status

All 28 chapters of the guide are written: ACE hub, print macros, bed mesh, calibration, core G-code, exclude object, extruder, factory commands, fans, filament sensor, heaters, homing, input shaper, LeviQ3, low-level motion, macros, misc, moves and modes, position/offsets/saved state, OTA, pause/resume/cancel, print job, print statistics, probe, queries, TMC drivers, timelapse, toolhead limits. Findings marked **(to verify)** are still open.

## What is not in this repository

Only the Markdown documents are tracked. The firmware files, the analysis scripts and data, the clone of the public source, and local test work (such as the sharp corners tuning files) are deliberately left out.
