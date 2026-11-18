# Bed Leveling Planner

`Bed Leveling Planner.xlsm` is an Excel workbook (with macros) that helps you flatten the bed of an **Anycubic Kobra 3 Max**. It reads the bed mesh from the printer, shows it as a heat map, and works out two physical corrections:

1. **Screw shims**: how thick a shim to put under each of the four bed screws to cancel the overall tilt of the bed.
2. **Kapton tape strips**: where to stick strips of tape under the flexible steel plate to fill in the low areas that the shims cannot remove.

It simulates the bed after each correction, so you can see the expected result before touching the printer. It can also run an **averaged mesh test** (several probings, averaged) to get a mesh that is not spoiled by the noise of a single measurement, and rates the quality of the measurement.

> **Read this before you click anything that says "Real test".** The real test homes the printer, heats the bed and nozzle, moves the toolhead, probes the bed several times and writes the printer's configuration file after every run. Only use it on an idle printer with a clear bed. See [Safety](#safety).

## Contents

- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Safety](#safety)
- [The workbook](#the-workbook)
- [The Parameters sheet](#the-parameters-sheet) (and the [constants in the code](#constants-in-the-code))
- [The buttons](#the-buttons)
- [How to apply the results](#how-to-apply-the-results)
- [How the calculations work](#how-the-calculations-work)
- [Assumptions and limits](#assumptions-and-limits)
- [Messages and troubleshooting](#messages-and-troubleshooting)
- [What was tested](#what-was-tested)

## Requirements

| What | Detail |
|---|---|
| **Excel** | Windows desktop Excel (2010 or later, 32 or 64 bit) with macros enabled. The macros use `MSXML2.ServerXMLHTTP` and the Windows `Sleep` function, so Excel for the web, Excel for Mac and LibreOffice do not work. |
| **Printer** | A Kobra 3 Max reachable over the network through **Moonraker** (default port 7125). The printer used for the development runs **Rinkhals**, which provides Moonraker; the workbook assumes its behaviour (see [Why the test goes through `M7000`](#why-the-test-goes-through-m7000)). |
| **Printer macros** | `M7000` (wraps a command so that Rinkhals does not rewrite it) is **required for the real test**. `M7001` (prints a message in the Mainsail console) is optional. Both are defined in [`../ORCA_SLICER_GCODE.md`](../ORCA_SLICER_GCODE.md) and must be in the printer configuration (the printer rebooted afterwards). |
| **Printer commands used** | `G28`, `MOVE_HEAT_POS`, `M140`, `M109`, `M190`, `G9112`, `M7000 BED_MESH_CALIBRATE`, `SAVE_CONFIG`, `TURN_OFF_HEATERS`, `M106 S0`, `M84`, `G9113`, `BED_MESH_PROFILE LOAD=default`, and `M7001`. They are described in [`../K3M_gcode_user_guide.md`](../K3M_gcode_user_guide.md). |
| **Bed** | A bed held by **four fixed screws** (shims, no leveling screws) and carrying a **flexible steel plate**. The mesh must be the stock **9 × 9** grid. |

Nothing else is needed: no Python, no add-in, no internet access.

## Quick start

1. Copy `Bed Leveling Planner.xlsm` to a local folder and open it. Click **Enable Content** (the buttons are macros).
2. Open the **Parameters** sheet and set **`PRINTER_URL`** to your printer, for example `http://192.168.1.46:7125`. Check the bed size and the screw positions (defaults: 420 × 420 mm, screws 50 mm from the sides and 70 mm from the front and back).
3. On the **Bed Leveling** sheet click **Get bed mesh**. It reads the mesh currently active in the printer (no motion, no heating) and fills in the three heat maps, the shim table and the tape strips. The banner on top shows the date, time and number of samples.
4. To work from a better measurement, **uncheck "Real test"** and click **Averaged mesh test** first: this is a harmless simulation (see [the buttons](#the-buttons)). When you are ready for the real thing, read [Safety](#safety), check the box and click again.
5. Read the **shim table** and the **strip table and drawing**, and compare the three heat maps (measured, with shims, with shims + tape).
6. Apply the corrections ([How to apply the results](#how-to-apply-the-results)), then measure again to check.

## Safety

- **The "Real test" checkbox is saved with the file.** The copy in this repository is saved with it **unchecked** (mock test), but Excel keeps whatever state you left, so look at the box before every click on *Averaged mesh test*: with the box checked the printer heats and moves (a confirmation window appears first, read it).
- **A real test needs an idle printer and a clear bed.** Do not start it during a print. The nozzle is wiped, then the toolhead moves over the bed at the probing temperature.
- **The real test overwrites your `default` mesh and writes the configuration file.** Every run calls `BED_MESH_CALIBRATE` without a profile name (so the result replaces the `default` mesh) and then `SAVE_CONFIG`, which writes `printer_mutable.cfg`. After the test, the stored `default` mesh is the mesh of the **last run**, not the average. If you want your previous mesh back, measure it again with a normal `BED_MESH_CALIBRATE`.
- **Never use named mesh profiles.** The workbook only ever uses `default`, on purpose: a saved named bed-mesh profile can lock the printer at the next start with **error 11407**. See the warning in chapter 3 of [`../K3M_gcode_user_guide.md`](../K3M_gcode_user_guide.md). The `TEST_PROFILE` constant in the code must stay `default` (see [Constants in the code](#constants-in-the-code)).
- **The Stop button does not interrupt a command already running on the printer** (a probing takes about 6 minutes). It ends the test at the next checkpoint and shuts the printer down.
- **Do not edit the sheet during a test.** Edits of the shim table are ignored until the test ends.
- **A failed or stopped real test shuts the printer down**: heaters off, cooling fan off, motors off. If the shutdown itself fails, a message tells you to check the printer by hand. Do that: a heater left on is the one real risk of this tool.

## The workbook

The workbook has two sheets: **Bed Leveling** (results and buttons) and **Parameters** (every setting).

### The Bed Leveling sheet

All matrices are drawn **as you see the bed from the front**: the last row is the front, the first column is the left side. All values are in millimetres.

| Area | Cells | Content |
|---|---|---|
| Banner | `A1` | Date, time and number of samples of the mesh on the sheet, and how it was obtained (`single read`, `averaged`, or `mock, simulated`). |
| **Measured mesh** | `C6:K14` | The 9 × 9 mesh read from the printer (the average of the runs after a test), rounded to two decimals, as a heat map (blue = lowest, red = highest). |
| Side tilts | `C4`, `K4` (back), `C16`, `K16` (front), `A6`, `A14` (left), `M6`, `M14` (right) | For each side, two numbers with opposite signs: half the height difference between the two corners of that side. Each is the value to add at that corner to level the side. |
| **Screw shims** | `P6:Q8` | Suggested shim thickness under the four screws (back-left `P7`, back-right `Q7`, front-left `P8`, front-right `Q8`). **You can edit these cells**: the simulation is recalculated. |
| **With screw shims** | `T6:AB14` | The measured mesh plus the tilt that the shims in `P7:Q8` would give (second heat map). |
| **With shims + tape** | `T19:AB27` | The same after the tape strips (third heat map). The three heat maps share **one colour scale** so they can be compared. |
| Test statistics | `P15:P18` | During and after an averaged test: variance between runs (mm²), its standard deviation (mm), the data-quality rating (`n/a` with a single run) and `Real` or `Mock`. |
| **Strip table** | `O30:V39` | One row per tape strip: the two ends `X1,Y1` and `X2,Y2` (mm from the front-left corner), length, angle and number of layers. |
| Summary | `P41:R43` | For *Original*, *Shims* and *+ Tape*: the peak-to-peak height range and the standard deviation of the 81 values. |
| **Tape drawing** | below the measured mesh | A top view of the bed (front at the bottom) with the four screws (grey dots), each strip, its number, size, angle, layers and the coordinates of its ends. The drawing is rebuilt on every calculation. |
| Buttons | top right | **Get bed mesh**, **Averaged mesh test**, the **Real test** checkbox, **Stop**, and the **+** and **−** buttons next to the strip list. |

### Reading the three heat maps

The point of the sheet is the comparison of the peak-to-peak and standard-deviation figures in the summary: *Original* is the bed as it is, *Shims* what the screws can do (they can only tilt the bed), *+ Tape* what the tape adds on top. A smaller range is flatter. The sheet gives you the numbers, not a verdict.

## The Parameters sheet

Every value is read **each time a button is pressed**, so there is nothing to save or recompile: edit the yellow cells, then click. An empty or invalid value stops the button with a message that names the parameter. Distances are in mm, temperatures in °C, and the origin (0, 0) is the **front-left corner** of the bed.

| Parameter | Default | Meaning |
|---|---|---|
| **Printer and bed** | | |
| `PRINTER_URL` | `http://192.168.1.46:7125` | Moonraker address, `http://ip:port`, no trailing slash. |
| `BED_W` | 420 | Bed width along X (at least 1). |
| `BED_D` | 420 | Bed depth along Y (at least 1). |
| `SCREW_DX` | 50 | Distance of the four shim screws from the left and right sides (at least 0). |
| `SCREW_DY` | 70 | Distance of the four shim screws from the front and back (at least 0). |
| `SHIM_STEP` | 0.05 | Thickness step of the shims you actually have. Shim sizes are rounded to a multiple of it (at least 0.001). |
| **Mesh test (real run)** | | |
| `TEST_BED_TEMP` | 55 | Bed temperature during the test. |
| `WIPE_TEMP` | 170 | Nozzle temperature while wiping (the stock `[leviQ3]` `extru_temp`). |
| `PROBE_TEMP` | 140 | Nozzle temperature while probing (the stock `[leviQ3]` `extru_end_temp`). |
| `SOAK_SECONDS` | 180 | Heat soak once the bed has reached its temperature. |
| `NUM_RUNS` | 5 | Number of probing runs (at least 1). With a single run the mesh is simply that run, and no variance or data-quality rating can be given (`n/a`). |
| `MOCK_SIGMA` | 0.02 | Standard deviation of the noise the mock test adds to each run. |
| `CONSOLE_MESSAGES` | Yes | `Yes` or `No`. `Yes` also prints each step of the test in the Mainsail console through `M7001`. |
| **Kapton tape** | | |
| `TAPE_W` | 20 | Width of a strip (at least 1). |
| `TAPE_T` | 0.065 | Thickness of one layer of tape, film plus adhesive (at least 0.001). |
| `TAPE_STOP` | 0.01 | Stop adding strips when the next one improves the spread by less than this fraction (0.01 = 1 %). |
| `MAX_STRIPS` | 5 | Number of strips to place at most, 1 to 8. Fewer are used when a strip no longer helps. The **+** and **−** buttons also change it. |
| `MAX_LAYERS` | 4 | Most layers of tape stacked in one strip (at least 1). |

### Constants in the code

Four values are deliberately **not** on the sheet. They are constants at the top of the VBA module (Alt+F11, `Module1`) and should not be changed unless you know why:

| Constant | Value | Meaning |
|---|---|---|
| `MESH_X0` | 10 | X of the first (left) mesh column, mm. |
| `MESH_Y0` | 10 | Y of the first (front) mesh row, mm. |
| `MESH_STEP` | 50 | Distance between two mesh points, mm. |
| `TEST_PROFILE` | `"default"` | Name of the mesh profile of the test. **Not used by any command today, and it must stay `"default"`.** |

The three mesh constants describe the printer's stock 9 × 9 mesh. They only feed the shim calculation, while the tape search hard-codes the same grid (5 mm steps from 10 to 410 mm), so changing one of them alone would make the shim and the tape calculations disagree.

> **⚠️ WARNING — never put a named bed-mesh profile in `TEST_PROFILE`.** A saved named profile (a `bed_mesh <name>` section in `printer_mutable.cfg`) can lock the printer at its next start with **error 11407**. The warning is repeated as a comment in the code, next to the constant. See the warning in chapter 3 of [`../K3M_gcode_user_guide.md`](../K3M_gcode_user_guide.md).

## The buttons

### Get bed mesh

Reads `bed_mesh` from Moonraker (`/printer/objects/query?bed_mesh`) and displays the **active** mesh. It sends no motion and no heating command, so it is safe at any time. The banner says `1 sample (single read)`.

Note that, on this printer, the mesh in `bed_mesh` is the one that has been **saved**: after a `BED_MESH_CALIBRATE` it only appears after `SAVE_CONFIG`.

### Averaged mesh test

Runs `NUM_RUNS` probings (one is enough) and uses their **average** for everything on the sheet, refreshing it after each run. From two runs on it also computes the variance between runs, which tells you how much to trust the mesh. What it does depends on the **Real test** checkbox.

**Real test unchecked (mock).** Reads the current mesh once and adds Gaussian noise (`MOCK_SIGMA`) to simulate each run, taking one second per run. Nothing is heated or moved, so it is safe even while the printer is busy. Use it to try the workbook and the Stop button.

**Real test checked.** After a confirmation window, it does this:

| Step | Command sent | Why |
|---|---|---|
| 1 | `G28` | Home. |
| 2 | `MOVE_HEAT_POS` | Park away from the bed before anything is heated, so that a drooling nozzle cannot drip on it. |
| 3 | `M140 S<TEST_BED_TEMP>`, `M109 S<WIPE_TEMP>` | Heat the bed and the nozzle. |
| 4 | `G9112` | Wipe the nozzle. |
| 5 | `MOVE_HEAT_POS` | Z to 15 mm, then X457, so that the nozzle does not wait on the wipe pad. |
| 6 | `M190 S<TEST_BED_TEMP>`, `M109 S<PROBE_TEMP>` | Wait for the bed, bring the nozzle to the probing temperature. |
| 7 | (wait) | Heat soak, `SOAK_SECONDS`, with a countdown in the status bar. |
| 8 | `M7000 BED_MESH_CALIBRATE`, then `SAVE_CONFIG`, once per run | Probe the whole bed, then save it so that it becomes visible. The mesh is read after each run, and the test waits up to 60 s for a mesh different from the previous one. |
| 9 | `TURN_OFF_HEATERS`, `M106 S0`, `M84`, `G9113`, `BED_MESH_PROFILE LOAD=default` | Shutdown: heaters off, cooling fan off, motors off, the "nozzle cleaned" memory reset, the `default` mesh reloaded. Every shutdown step is tried even if one fails. |

With the stock 9 × 9 mesh a probing takes about 6 minutes, so a 5-run test lasts about **40 minutes** with the heating and the soak (an estimate, not a measured figure).

While it runs, Excel is **not frozen**: requests are asynchronous and the macro gives control back to Excel every 50 ms. The status bar shows the progress, and each step is also printed in the Mainsail console when `CONSOLE_MESSAGES` is `Yes` (the mock test prints them too). If the printer has no `M7001` macro the messages are silently dropped and the test goes on. If it has no `M7000` macro, a real test stops **before** sending anything.

**Result, updated after every run.** Whatever the number of runs, the sheet is refreshed after each run with the **average of the runs collected so far**: the mesh, the shims, the tape strips, the three heat maps, the summary and the statistics. The banner shows how far you are, for example `3 samples (averaged so far, 5 runs planned)`; after the last run it reads `5 samples (averaged)`. Refreshing takes a few seconds (the tape search is the slow part), during which the printer waits before the next run starts. Early on, the figures are those of few runs and move as more are added; the point of seeing them is to notice a bad probing early and press **Stop**. The statistics are:

- the **variance between runs**: the sample variance of each of the 81 points over the runs, averaged over the points (mm²), and its square root. With only one run so far there is no variance: the cells show `n/a`;
- a **data-quality rating** from that standard deviation: **Excellent** ≤ 0.01 mm, **Good** ≤ 0.02, **Fair** ≤ 0.04, **Poor** above (`n/a (1 run)` for a single run).

A Poor rating means the probing itself is noisy (loose bed, dirty or too cold nozzle, insufficient soak): fix that before trusting any shim or tape advice.

### Stop

Ends the running test at the next checkpoint: between two printer commands, or within a second during the heat soak. A command already running on the printer is **not** interrupted; the macro waits for it, then shuts the printer down. The sheet keeps the average of the runs completed before the stop (a message says how many), and the banner shows `k samples (averaged so far, n runs planned)`, so a stopped test is recognisable as partial. The same applies if a test fails.

### + and −

Add or remove **one tape strip** relative to the strips currently listed, keeping the current mesh and shims, and recompute the drawing and the third heat map. The number is also written to `MAX_STRIPS`. **+** adds the next best strip **even if it gains less than `TAPE_STOP`**, and tells you when no strip reduces the spread any more. Editing a shim afterwards recomputes with the normal stop rule again.

### Editing the shim table

Typing a thickness in `P7:Q8` recalculates the "with shims" and "with shims + tape" maps, so you can try the shims you really have. It is ignored while a test is running.

### Why the test goes through `M7000`

On a Rinkhals printer, Moonraker (`kobra.py`, `patch_bed_mesh`) **replaces** any `gcode/script` that starts with `BED_MESH_CALIBRATE` by a whole leveling sequence: move to the heating position, bed and nozzle to the `[leviQ3]` temperatures, wipe, a bare `BED_MESH_CALIBRATE`, `TURN_OFF_HEATERS`, `M106 S0` and `SAVE_CONFIG`. Sent as it is, the command would heat and wipe again before every run, turn the heaters off after each probing and overwrite the `default` mesh. Wrapped in `M7000` the command reaches the firmware unchanged. (`PROFILE`, `PROBE_COUNT` and `MESH_MIN`/`MESH_MAX` are not sent: the firmware ignores the last two and a named profile is dangerous; see the guide.)

## How to apply the results

1. **Check the sign convention first** (see [limits](#assumptions-and-limits)): the workbook assumes that a **higher mesh value means the bed is higher** at that point.
2. **Shims.** The table gives the thickness to **add** under each of the four screws (back-left, back-right, front-left, front-right). The smallest value is always 0: the other screws are raised relative to it. Fit shims of the nearest sizes you have (set `SHIM_STEP` to your smallest shim increment so that the figures are multiples of it).
3. **Tape.** Each strip row gives the two ends in mm from the **front-left corner** of the bed (X to the right, Y to the back), the length, the **angle** (from the X axis toward the back, in 10° steps) and the number of **layers**. Stick the strips **under the flexible plate**, on the bed, as drawn. Strips are `TAPE_W` wide (20 mm by default); a strip of several layers means that many layers stacked on top of each other along the same line.
4. **Measure again**: click **Get bed mesh** (after a normal `BED_MESH_CALIBRATE` and `SAVE_CONFIG`) or run a new averaged test, and compare with the prediction. The summary table of the new run tells you whether you are done or should iterate.

## How the calculations work

All of it is in the VBA module (Alt+F11, `Module1`).

**Display.** The 81 mesh values are written with the front on the last row, rounded to two decimals. Heat-map colours go from blue (lowest) through cyan, green and yellow to red (highest), over the minimum and maximum of the three matrices together.

**Side tilts.** For each side, half the difference between its two corner values, with opposite signs at the two ends.

**Shims.** The bed is treated as **rigid**: shims can only tilt it, not remove its warp.
1. A least-squares plane is fitted through the 81 points (slopes in X and Y around the mesh centre).
2. The shim at each of the four screws is minus the height of that plane at the screw, shifted so that the smallest is 0, then rounded to a multiple of `SHIM_STEP`.
3. The simulated bed is the measured mesh plus the plane made by the (rounded or edited) shims. A twist between the four shims cannot be applied to a rigid bed, so the plane used is the least-squares one through the four shim heights.

**Tape.** The flexible plate follows the surface, so tape can fill in local lows. Starting from the "with shims" surface:
1. The surface is interpolated bilinearly on an **81 × 81 grid with a 5 mm step**.
2. For every strip position, angle (every 10°, from 0° to 170°), length (5 mm step) and number of layers (1 to `MAX_LAYERS`), the **gain** is the reduction of the sum of squared deviations from the mean surface height. For a strip of thickness *t* over a point at deviation *r* the gain is −(2·r·*t* + *t*²), so tape helps only on points below the mean. For each angle and offset, the best length is found with a best-run search along the strip (Kadane's algorithm).
3. The best strip is added to the surface after a 1-2-1 smoothing (one 5 mm step in X then Y): the plate is flexible and does not step sharply over the edge of a strip.
4. This repeats, strip after strip (so the first strips never change when you add more), until `MAX_STRIPS` is reached or the best remaining gain is below `TAPE_STOP` times the current sum of squares. The **+** button ignores the `TAPE_STOP` rule once.

**Statistics.** Peak-to-peak (maximum minus minimum) and standard deviation of the 81 values of each of the three matrices (computed on the rounded values shown). The test statistics are computed on the unrounded values of the runs.

## Assumptions and limits

- **Sign convention (to verify on your printer).** A higher mesh value is assumed to mean a higher bed. If your firmware uses the opposite sign, the shims belong on the opposite corners and the tape on the opposite areas.
- **Orientation.** Mesh row 0 is the front and column 0 is the left side, seen from the front.
- **Fixed 9 × 9 mesh.** The mesh is read as exactly 81 values; any other size stops with an error. The tape search assumes the points are 10 mm to 410 mm in 50 mm steps (the `MESH_X0`, `MESH_Y0` and `MESH_STEP` constants of the code, see [Constants in the code](#constants-in-the-code)). The firmware ignores `PROBE_COUNT`, `MESH_MIN` and `MESH_MAX` anyway.
- **The tape simulation is optimistic.** A stiff real plate may bridge narrow strips instead of following them; the 1-2-1 smoothing (about 5 mm) is a guess. Use the tape result as a starting point and re-measure.
- **The shims cannot remove warp**, only tilt. Whatever is left after the shims is what the tape is asked to fix.
- **Moonraker only.** Every read and command goes through Moonraker on the printer, and the printer commands (`G9112`, `MOVE_HEAT_POS`, `M7000`) have to exist.
- **Calculations are a simulation**, not a measurement. Only a new mesh shows what the corrections really did.

## Messages and troubleshooting

| Message | Meaning and fix |
|---|---|
| `Parameter X not found on the Parameters sheet` | A defined name `prm_X` was deleted or a row was removed. Restore the row or undo. |
| `Parameter X is empty` / `must be a number` / `must be at least N` | Fix the yellow cell named in the message. |
| `MAX_STRIPS is at most 8` | The strip arrays hold 8 strips: set `MAX_STRIPS` between 1 and 8. |
| `No answer from the printer (…)` | Wrong `PRINTER_URL`, printer off, or Moonraker down. Open the address in a browser. |
| `HTTP 404/500: …` | Moonraker answered with an error, usually a command the printer does not know. The message holds the printer's text. |
| `probed_matrix not found` / `expected 81 mesh values, got N` | No mesh is active, or the mesh is not 9 × 9. Load or measure one. |
| `The printer has no M7000 macro` | Install `M7000` (see [`../ORCA_SLICER_GCODE.md`](../ORCA_SLICER_GCODE.md)) and reboot the printer. Nothing was sent. |
| `The mesh did not change in the 60 s after the probing` | The probing ran but its result did not become the active mesh (the `SAVE_CONFIG` did not take effect). The message includes the active profile name. A mesh read identical to the previous one also stops the test, because it would mean the probing did not run. |
| `A mesh test is running` | Wait for it or press **Stop**. |
| `Read a bed mesh first` | **+** / **−** need a mesh on the sheet: click **Get bed mesh**. |
| `No further strip reduces the spread of the surface` | **+** found nothing more to add; the count is reset to the strips shown. |
| `Data collected, but the shutdown failed. Check the printer` | The result is kept but a shutdown command failed: **check that the heaters and motors are off by hand.** |
| No messages in the Mainsail console | The printer has no `M7001` macro, or `CONSOLE_MESSAGES` is `No`. The test is not affected. |
| Buttons do nothing | Macros are disabled: click **Enable Content**, or unblock the file (right-click, Properties, Unblock) if it was downloaded. |

## What was tested

- A real averaged test of 5 runs was run on the author's printer; the saved sheet shows its result (mesh of 2026-11-17 23:22, 5 samples, averaged). The strip drawing, the shim table, the statistics and the Stop and **+** / **−** buttons are part of that version.
- The `M7000` rewrite problem, the need for `SAVE_CONFIG` before the new mesh becomes visible, and the fact that `PROFILE`, `PROBE_COUNT` and `MESH_MIN`/`MESH_MAX` are ignored were all observed on the printer (see chapter 3 of [`../K3M_gcode_user_guide.md`](../K3M_gcode_user_guide.md)).
- **Not verified**: that fitting the suggested shims and strips improves the first layer as predicted. Treat the output as advice to check, not a promise.
- The one-run test and the refresh after every run were checked with simulated runs (no printer): one run shows `n/a` for the variance and rating, a partial average shows `k samples (… so far, n runs planned)`, and the final state equals the full average. They have **not** been run in a real test on the printer.
