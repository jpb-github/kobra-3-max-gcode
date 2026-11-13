# Anycubic Kobra 3 Max — G-code and macro guide

A user-friendly guide to the G-code commands and macros of the Anycubic Kobra 3 Max
(firmware **2.5.2.8**, "Klipper-Go"), written from an analysis of the firmware binary
(`gklib`). The public Klipper-Go source on GitHub is out of date, so it was used only
as a hint, never as the source of truth.

> **Nothing in this guide has been test-run on a printer.** Statements that rest on
> inference are marked **(to verify)**. Do not send G-code to your printer on the
> strength of this guide alone.

## Contents

| File | What it is |
|---|---|
| [`K3M_gcode_user_guide.md`](K3M_gcode_user_guide.md) | **The guide.** One chapter per command family: what each command does step by step, every parameter (meaning, values, default), examples, cautions. |
| [`K3M_2.5.2.8_gcode_commands.md`](K3M_2.5.2.8_gcode_commands.md) | Technical reference: the full table of the 175 registered commands, the extraction method and key findings. |
| [`PROGRESS.md`](PROGRESS.md) | Work log and bootstrap notes: chapter status, findings, open questions, how to resume the analysis. |

## Status

Chapters 1–19 of 28 are written (ACE hub, print macros, bed mesh, calibration, core
G-code, exclude object, extruder, factory commands, fans, filament sensor, heaters,
homing, input shaper, LeviQ3, low-level motion, macros, misc, moves and modes,
position/offsets/saved state). Next: OTA, pause/resume/cancel, print job, print stats,
probe, queries, TMC drivers, timelapse, toolhead limits. See `PROGRESS.md` for details.

## What is not in this repository

Only the Markdown documents are tracked. The firmware files, the analysis scripts and
data, and the clone of the public source are deliberately left out.
