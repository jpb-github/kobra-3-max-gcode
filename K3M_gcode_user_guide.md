# Anycubic Kobra 3 Max — G-code & Macro Guide

*Applies to firmware 2.5.2.8. Written from an analysis of the firmware itself (not from the published source code, which is out of date). Commands were not test-run on a printer unless stated; behaviour marked "(to verify)" is the least certain part.*

## How to read this guide

- **Syntax** shows the command with its parameters. `[ ]` means optional, `< >` means "replace with your value". Do not type the brackets.
- Parameters are written `NAME=value` (for example `BEDTEMP=60`). Commands are not case-sensitive, and the parameter names listed here are the ones the printer actually reads. Anything else you add is ignored.
- Temperatures are in °C, distances in mm, speeds in mm/min unless stated.
- You can send these commands from the console of Mainsail/Fluidd (the printer's web interface), from a G-code file, or from a macro.

---

# Chapter 1 — ACE filament hub

These commands drive the ACE multi-colour filament hub (feeding, retracting, cutting and swapping filaments). They are mostly called by the slicer's start/end code and by the touchscreen, rather than typed by hand. This chapter is a summary only.

| Command | Parameters (default) | What it does |
|---|---|---|
| `ACE_INFO` | `ID` (0) | Reports the hardware/identification details of the ACE unit number `ID`, as a JSON text. Reports an error if the unit cannot be found. |
| `ACE_STATUS` | — | Reports the current state of the whole hub (units, slots, loaded filament…) as a JSON text. |
| `T<n>` (`T0`, `T1`, …) | — | Tool change: unloads the current filament, loads the filament assigned to tool `n`, then purges the old colour out of the nozzle. Needs the printer homed. |
| `FEED_FILAMENT` | `ID` (0), `INDEX` (0) | Loads the filament of slot `INDEX` of ACE unit `ID` all the way to the nozzle (heats the nozzle if needed, feeds, purges) and marks it as the current filament. |
| `UNWIND_FILAMENT` | `ID` (0), `INDEX` (0) | Pulls the filament of that slot back out of the printer into the hub. |
| `UNWIND_ALL_FILAMENT` | — | Cuts and retracts every loaded filament back into the hub (lifts Z first; homes X if it is not homed). |
| `UNWIND_CUT_FILAMENT` | — | Cuts the filament with the cutter, then retracts it. |
| `CUT_FILAMENT` | — | Cuts the filament with the hub's cutter, leaving it in place. |
| `SET_CURRENT_FILAMENT` | `ID` (0), `INDEX` (0) | Tells the printer which slot is currently loaded in the nozzle. Nothing moves; use it to resynchronise after a manual change. |
| `EXTRUDE_FILAMENT` | `LENGTH` (0), `SPEED` (0) | Pushes `LENGTH` mm of filament through the nozzle at `SPEED` and discards what came out. Both are whole numbers. |
| `CLEAN_PIPE` | — | Purges the filament path: sets the nozzle temperature, goes to the purge position (homing X/Y if needed), pushes the remaining filament out and discards it. Motors are released at the end. |
| `REFILL_FILAMENT` | — | Automatic spool swap: when a spool runs out, finds another spool of the same colour, loads it and continues. If none is available the print is paused. |
| `RUNOUT_PAUSE` | — | Pauses the print because the filament ran out, after cleaning the filament path. Normally triggered by the filament sensor, not typed by hand. |
| `CYCLE_FEED_FILAMENT` | — | Exists in the firmware; its purpose could not be determined (not documented). |
| `OTA_FILAMENT_HUB_START` | `UPDATE_PATH` (`./ACE_V2.0.1_20240104.bin`), `ID` (0) | Flashes new firmware into ACE unit `ID` from the given file. **Only use with an official firmware file; do not interrupt it.** |

---

# Chapter 2 — Anycubic print macros (`G9111`, `G9112`, `G9113`)

These three commands are the "get ready to print" sequence that Anycubic's slicer profiles put at the start of a print: clean the nozzle, measure the bed/nozzle height automatically (LeviQ 3), bring everything to printing temperature. They work together, and they share one piece of memory: **"the nozzle has already been cleaned"**.

| Command | In one line |
|---|---|
| `G9111` | The full pre-print sequence: heat, clean the nozzle, auto-measure the Z offset, go to print temperature. |
| `G9112` | Only the nozzle cleaning part (wipe on the silicone wiper). Runs at most once until reset. |
| `G9113` | Resets the memory "nozzle already cleaned", so that `G9112` is allowed to run again. |

**How they fit together.** `G9112` cleans the nozzle and remembers that it did. `G9111` checks that memory: if the nozzle was *not* cleaned yet, `G9111` heats up and calls `G9112` itself; if it was already cleaned (for example your start code called `G9112` first), `G9111` skips that part and goes straight to the Z-offset measurement. When `G9111` finishes, or the print is cancelled, the memory is reset.

> **Tip.** All three assume a working LeviQ 3 setup (the `[leviQ3]` section of `printer.cfg`) and the `WIPE_NOZZLE`, `MOVE_HEAT_POS` and `UNDERLINE` macros of the stock configuration. If you changed those macros, the behaviour of `G9111`/`G9112` changes accordingly.

---

## `G9111` — Full automatic pre-print sequence

### What it does

`G9111` prepares the printer for printing and measures the nozzle-to-bed height automatically. Step by step:

1. **Clears a previous pause state** (like `CLEAR_PAUSE`), so a leftover pause does not interfere.
2. **Cleans the nozzle — only if it has not been cleaned yet** (see "How they fit together" above). When it has to do this, it:
   1. starts heating the bed to the *bed temperature* without waiting;
   2. heats the nozzle to the *wipe temperature* and waits for it;
   3. runs `G9112` (wipes the nozzle);
   4. waits for the bed to reach its temperature;
   5. lowers the nozzle to the *probing temperature* (140 °C on the stock configuration) and waits, so that the nozzle does not ooze while it touches the bed.
3. **Measures the Z offset automatically** (`LEVIQ_AUTO_ZOFFSET`). To do so it first remembers which bed-mesh profile is loaded, clears the mesh, runs the measurement and waits for all moves to end, then **loads the remembered mesh profile again** (or the profile named `default` if none was loaded).
4. **Moves the toolhead to the heating position** (`MOVE_HEAT_POS`: Z 15, then X 457 on the stock configuration).
5. **Heats the nozzle to `EXTRUDERTEMP` and waits** — the temperature you want to start printing at.
6. **Applies the temperature compensation of the Z offset** (`LEVIQ_TEMP_OFFSET`).
7. **Draws the purge line** (`UNDERLINE` macro) — only when the ACE filament hub is in use for this print *(verified from the firmware's internal "no ACE hub" flag: with no ACE connected the step is skipped)*. Without ACE, your slicer's own start code is expected to purge.
8. **Clears the message on the screen** (`M117`) and **resets** the "nozzle already cleaned" memory.

If the print is cancelled while `G9111` runs, the sequence stops at the next step.

### Syntax

```
G9111 [EXTRUDERTEMP=<°C>] [BEDTEMP=<°C>] [WIPETEMP=<°C>]
```

### Parameters

All parameters are optional. They must be written as `NAME=number` with **no spaces** around the `=`. Names are not case-sensitive (`bedtemp=60` works). A word without `=` or a value that is not a number makes the command fail.

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `EXTRUDERTEMP` | Nozzle temperature the printer waits for at the end of the sequence (step 5), i.e. the temperature at which the print is about to start. | Any temperature in °C within the heater's limits. | `220` |
| `BEDTEMP` | Bed temperature used while measuring (steps 2.1 and 2.4). | Any temperature in °C within the bed's limits. **Ignored if the file being printed declares its own bed temperature** — the temperature written in the G-code file's slicer header (`first_layer_bed_temperature` / `bed_temperature`) then wins *(to verify)*. | `60` |
| `WIPETEMP` | Nozzle temperature used while the nozzle is wiped (step 2.2). | Any temperature in °C. | The `extru_temp` value of the `[leviQ3]` section of `printer.cfg` (**170** on the stock configuration; 190 if the option is missing). |

The temperature the nozzle is lowered to for probing (step 2.5) is **not** a parameter: it is the `extru_end_temp` setting of the `[leviQ3]` section (default and stock value: 140).

### Examples

```
G9111
```
Run the whole sequence with the defaults (bed 60 °C, wipe at the configured 170 °C, start printing at 220 °C).

```
G9111 EXTRUDERTEMP=250 BEDTEMP=100
```
Typical for ABS: bed at 100 °C for the measurement, nozzle at 250 °C when the sequence ends.

```
G9111 EXTRUDERTEMP=215 BEDTEMP=60 WIPETEMP=160
```
Wipe at a lower temperature (less oozing on soft filaments), start at 215 °C.

Typical start code in a G-code file:

```
G28                       ; home all axes
G9111 EXTRUDERTEMP=210 BEDTEMP=60
; ... the rest of your start code (purge line, etc.) ...
```

### Notes and cautions

- **Home first** (`G28`). `G9111` only homes by itself when it runs the cleaning part; when the cleaning was already done, the Z-offset measurement assumes the printer is homed.
- `G9111` changes the heater targets and leaves the nozzle at `EXTRUDERTEMP` and the bed at the bed temperature when it ends.
- The measurement changes the stored Z offset (LeviQ 3). It is meant to be run before every print and can be disabled with `LEVIQ_AUTO_ZOFFSET_ON_OFF` (covered in the LeviQ 3 chapter).
- If the printer is cancelled during the sequence, nothing needs to be cleaned up manually: the "already cleaned" memory is reset automatically.

---

## `G9112` — Clean the nozzle on the wiper

### What it does

Wipes the nozzle on the silicone wiper pad at the rear of the bed. It does **not** heat anything: the nozzle must already be at a suitable temperature (`G9111` does that for you). Step by step:

1. **Checks the memory "nozzle already cleaned".** If it is set, it does nothing at all except write a warning in the log (`nozzle clear cnt > 0, return`) — so calling `G9112` twice in a row is harmless but the second call does nothing.
2. **Remembers the loaded bed-mesh profile and clears the mesh**, so that the wiping moves are not bent by the mesh compensation.
3. **Homes the printer if needed**: if X, Y and Z are not all homed, it runs `G28 W` first.
4. **Resets the speed and acceleration limits** to their normal values (`RESET_VELOCITY_LIMIT`).
5. **Switches the X and Y motor drivers out of silent mode** (StealthChop off), waits 0.1 s.
6. **Wipes the nozzle** with the `WIPE_NOZZLE` macro (part-cooling fan at 100 %, then three passes of the nozzle across the wiper pad — on the stock configuration at the left of the rear edge, X ≈ 42–102, Y ≈ 428–430) and waits for the moves to finish.
7. **Switches StealthChop back on** for X and Y, waits 0.1 s.
8. **Loads the bed-mesh profile again** (the one remembered in step 2, or `default` if none was loaded).
9. **Sets the memory "nozzle already cleaned".**

If the print is cancelled while the wipe is running, the command is interrupted and the memory is reset.

### Syntax

```
G9112
```

### Parameters

None. Anything typed after the command is ignored.

### Examples

```
M109 S170        ; heat the nozzle to wiping temperature first
G9112            ; wipe the nozzle
```

```
G9113            ; allow another wipe
G9112            ; wipe again (otherwise this call would be skipped)
```

### Notes and cautions

- **The nozzle is hot and the wiping area is at the back of the bed**: keep the area clear and do not run the command with a cold nozzle (it would drag solid plastic across the pad).
- The wiping positions and the number of passes belong to the macros `WIPE_NOZZLE` / `SINGLE_WIPE_NOZZLE` in `printer.cfg`; change them there, not in `G9112`.
- Run twice in the same session only after `G9113` (or after a cancelled print / finished `G9111`, which reset the memory).

---

## `G9113` — Allow the nozzle to be cleaned again

### What it does

Resets the memory "the nozzle has already been cleaned". Nothing moves, nothing heats, nothing is displayed. After `G9113`, the next `G9112` (or the next `G9111`) will clean the nozzle again.

You rarely need it by hand: `G9111` resets the memory itself at the end of the sequence, and cancelling a print resets it too.

### Syntax

```
G9113
```

### Parameters

None.

### Examples

```
G9113      ; forget that the nozzle was cleaned
G9111      ; full sequence, including the nozzle wipe
```
Forces `G9111` to include the cleaning step even if `G9112` already ran earlier.

---

# Chapter 3 — Bed mesh

The bed is never perfectly flat. The **bed mesh** is a grid of height measurements ("probed points") taken over the bed. While printing, the printer uses it to raise or lower the nozzle slightly so that the first layer follows the real surface of the bed.

Key ideas used in this chapter:

- **Mesh**: the grid of measured heights currently in use. There is at most one active mesh at a time.
- **Profile**: a named copy of a mesh kept in memory, so that you can reload it later without measuring again. The stock printer uses a profile called `default`. **Use only that one: other names can lock the printer (see the warning below).**
- **Fade**: the correction is applied fully on the first layers and gradually removed as the nozzle goes up, so a tall print is not warped by the bed's shape. On the stock configuration the correction is full at Z = 1 mm and fully gone at Z = 10 mm (`fade_start` = 1, `fade_end` = 10, `fade_target` = 0 in the `[bed_mesh]` section of `printer.cfg`).
- **Saved to disk or not**: measuring a mesh or saving a profile only keeps it **for the current session**. To keep it after a restart you must run `SAVE_CONFIG` (which writes the configuration file **without restarting** the firmware — its help text and the bed-mesh message still say "restart" — see the note on where the values land and what the status objects show at the end of this chapter's `SAVE_CONFIG` remark below).

The stock configuration measures a 9 × 9 grid (81 points) between X/Y = 10 and 410 mm, and smooths it with a "bicubic" interpolation.

> ## ⚠️ WARNING — never use named bed-mesh profiles (printer locked with error 11407)
>
> **Use only the `default` mesh.** Never run `BED_MESH_CALIBRATE PROFILE=<name>` or `BED_MESH_PROFILE SAVE=<name>` with another name, and never let a `SAVE_CONFIG` write such a profile.
>
> A saved named profile (a `"bed_mesh <name>"` section in `printer_mutable.cfg`) can **lock the printer at its next start with error 11407**. The firmware reads that section as a *configuration* section, stops with `Option 'probe_count' in section 'bed_mesh <name>' must be specified`, and the printer program (`gklib`) exits. The touchscreen stays on the error message, which hides every button (a USB firmware update cannot be started from it either), and the SSH and web interfaces did not answer. Seen on 2026-11-18 with a profile named `shimtest`. The only name seen to survive restarts is `adaptive` (created by an adaptive-mesh print start): that is an observation, not a guarantee.
>
> **If it already happened — principle of the fix.** The printer has a USB mechanism that needs no screen and no network: at start-up it looks for `update.swu` in the folder `aGVscF9zb3Nf` of a FAT32 USB stick, unpacks it (a password-protected zip, the same "SWU" password as the official firmware packages) and runs the `update.sh` it contains as root. A package whose `update.sh` replaces `/userdata/app/gk/printer_mutable.cfg` by a copy **without** the `bed_mesh <name>` section (keeping the old file as a backup, removing `update.swu` from the stick, then rebooting) brings the printer back; one beep means the package was found, two beeps mean the script ran. The other settings stored in that file (PID, Z offset, input shaper, `default` mesh…) are kept, which is why you should have a copy of it (the USB "export configuration" feature of the printer produces one).

| Command | In one line |
|---|---|
| `BED_MESH_CALIBRATE` | Measures the bed and creates the mesh (and saves it as a profile). |
| `BED_MESH_PROFILE` | Saves, loads or deletes named mesh profiles. |
| `BED_MESH_CLEAR` | Switches the mesh off (no height correction). |
| `BED_MESH_OUTPUT` | Prints the current mesh, or the list of points that will be measured, in the console. |
| `BED_MESH_MAP` | Prints the current mesh in a machine-readable form (used by the web interface's heat-map). |
| `BED_MESH_OFFSET` | Meant to shift the mesh in X/Y — **does not work in this firmware** (see below). |

---

## `BED_MESH_CALIBRATE` — Measure the bed

### What it does

1. **Switches off the current mesh** (like `BED_MESH_CLEAR`), so that the measurement is made on the uncorrected bed.
2. **Reads your parameters** and builds the list of points to measure (for the stock setup, 9 × 9 points from X/Y 10 to 410).
3. **Measures every point** with the printer's probe: the toolhead travels to each point and touches the bed, one after the other. This takes a while for 81 points.
4. **Builds the mesh** from the measurements (interpolating between the measured points with the chosen algorithm) and **activates it**.
5. **Stores the mesh in a profile** with the name given in `PROFILE` (`default` if you give none). This replaces any profile of the same name *for the current session*. Run `SAVE_CONFIG` afterwards if you want to keep it after a restart.

The command does not heat anything and does not home the printer. **Home first (`G28`)**, bring the bed to the temperature you will print at, and make sure the nozzle is clean (see `G9112`) — a blob of plastic on the nozzle ruins the measurement.

### Syntax

```
BED_MESH_CALIBRATE [PROFILE=<name>] [MESH_MIN=<x>,<y>] [MESH_MAX=<x>,<y>]
                   [PROBE_COUNT=<n>[,<m>]] [ALGORITHM=lagrange|bicubic]
                   [RELATIVE_REFERENCE_INDEX=<index>]
                   [ADAPTIVE=0|1] [ADAPTIVE_MARGIN=<mm>]
```

(Type it on one line. Two more options, `MESH_RADIUS`/`MESH_ORIGIN`/`ROUND_PROBE_COUNT`, exist for round beds and are not useful on this printer — see below.)

### Parameters

All optional. Whatever you leave out comes from the `[bed_mesh]` section of `printer.cfg`. Values given on the command line apply **only to this measurement**; they are not stored in the configuration.

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `PROFILE` | Name under which the new mesh is stored. **Do not use it**: a name other than `default` can lock the printer with error 11407 (see the warning at the start of this chapter). | Leave it out (= `default`). | `default` |
| `MESH_MIN` | X,Y position of the first (front-left) point of the grid — the corner of the measured area with the smallest coordinates. | Two numbers separated by a comma, **no spaces**: `MESH_MIN=20,20`. Must lie within the bed. | `10,10` (from `printer.cfg`) |
| `MESH_MAX` | X,Y position of the last (rear-right) point of the grid. | Same format: `MESH_MAX=400,400`. Must be larger than `MESH_MIN`. | `410,410` (from `printer.cfg`) |
| `PROBE_COUNT` | Number of points to measure along X and Y. | `n,m` (X count, Y count) or a single number `n` (used for both). Each count is **at least 3**. | `9,9` (from `printer.cfg`) |
| `ALGORITHM` | How the values between the measured points are calculated. | `bicubic` (smoother) or `lagrange`. See the rules below. | `bicubic` (from `printer.cfg`) |
| `RELATIVE_REFERENCE_INDEX` | Number of the measured point whose height is taken as "zero" for the whole mesh. The point numbers are shown by `BED_MESH_OUTPUT PGP=1`. | A whole number from 0 to (number of points − 1). | Not used (no reference point) |
| `ADAPTIVE` | Measure only the area where the model's first layer will be printed, instead of the whole bed. See "Adaptive mesh in detail" below — it only has an effect **during a print started from an Anycubic Slicer Next file**; otherwise the whole bed is measured, **without any message**. | `0` = whole bed. Any other whole number (use `1`) = adaptive. | `0` |
| `ADAPTIVE_MARGIN` | Extra border, in mm, added around the first-layer area when `ADAPTIVE=1`. Ignored otherwise. | A number ≥ 0. | The `adaptive_margin` value of `[bed_mesh]` in `printer.cfg` (0 if absent — it is absent on the stock configuration) |

> **Observed on the printer (2026-11-17, command sent through Moonraker `/printer/gcode/script`, wrapped in the `M7000` macro of `ORCA_SLICER_GCODE.md` because of the next note):**
>
> - **`PROBE_COUNT`, `MESH_MIN` and `MESH_MAX` do not work.** `M7000 BED_MESH_CALIBRATE PROFILE=shimtest PROBE_COUNT=3,3 MESH_MIN=100,100 MESH_MAX=300,300` (the `shimtest` profile it left is what locked the printer with error 11407: do not repeat this) measured the **whole 9 × 9 grid** (X/Y 10 to 410, 81 points, about 6 minutes), and the profile that was stored has `x_count`/`y_count` 9 and `min`/`max` 10 to 410. Treat the grid as fixed by `[bed_mesh]`.
> - **`PROFILE` works, but the result is invisible until `SAVE_CONFIG`.** When the measurement ends the command returns without an error and `bed_mesh` (Moonraker `printer/objects/query?bed_mesh`, `BED_MESH_PROFILE LOAD=…`) still shows the **previous** mesh. Only after `SAVE_CONFIG` do the new mesh and the new profile appear (a `bed_mesh shimtest` section in `printer_mutable.cfg`, and `default` with the new values after a run without `PROFILE`). Before that, `BED_MESH_PROFILE LOAD=shimtest` changes nothing and prints no error. So the mesh is measured and staged in memory, and the status object and the profile list show only the saved configuration. Reading the mesh right after the measurement therefore returns the old one.
> - In a print started from a file the adaptive form was seen to work (`M7000 BED_MESH_CALIBRATE ADAPTIVE=1 ADAPTIVE_MARGIN=5 PROFILE=adaptive` left a `bed_mesh adaptive` profile of 8 × 8 points in `printer_mutable.cfg`, probably saved by a later `SAVE_CONFIG` of the print start), so `ADAPTIVE` is not covered by the first point.
>
> **A bare `BED_MESH_CALIBRATE` sent through Moonraker on a Rinkhals printer is not what runs.** Rinkhals' Moonraker (`kobra.py`, `patch_bed_mesh`) replaces any `gcode/script` that starts with `bed_mesh_calibrate` (any case, any parameters, which are dropped) by `MOVE_HEAT_POS`, `M140`/`M109`/`M190` to the `[leviQ3]` temperatures (60 / 170 °C), `WIPE_NOZZLE`, `M109` to `extru_end_temp` (140 °C), a **bare** `BED_MESH_CALIBRATE`, `TURN_OFF_HEATERS`, `M106 S0` and `SAVE_CONFIG`. Seen in the console log as one multi-line command with the `G28` before it. Consequences: the heaters are turned off afterwards, the mesh is stored as `default` and written to `printer_mutable.cfg` each time. Starting the command with another word (`M7000 BED_MESH_CALIBRATE …`) avoids the replacement. Typed in the Mainsail console it most likely goes through the same path **(to verify)**.

**Rules for `PROBE_COUNT` and `ALGORITHM`** (the printer enforces them and stops with an error when they are broken):

- `lagrange` accepts at most **6 points** per axis.
- `bicubic` needs at least **4 points** per axis; with fewer, the printer silently switches to `lagrange` (so you must then respect the limit of 6).
- `bicubic` does not accept "3 points on one axis with more than 6 on the other".
- With `ADAPTIVE=1` the number of points is reduced in proportion to the area measured (see below), and the usual limits above still apply.

**Adaptive mesh in detail (`ADAPTIVE=1`)**

*Where the information comes from.* While a print is running, the printer reads the whole G-code file in the background (started when the print starts) and looks for four comment lines, each at the very beginning of a line and written exactly like this:

```
; print_bed_min = 0,0
; print_bed_max = 426,420
; first_layer_print_min = 106.471,89.7252
; first_layer_print_max = 358.116,341.48
```

Anycubic Slicer Next writes them from its machine start G-code (the stock profile's start code is `G9111 bedTemp=… extruderTemp=…`, then `M117`, then these lines). Files from other slicers (for example the sample Benchy files, which come from a different slicer) do not contain them. All four lines must be present and valid; the first-layer rectangle is what is used.

*What it does with them.*
1. The measured area becomes the rectangle of the first layer, enlarged by `ADAPTIVE_MARGIN` on every side, and **never larger than the normal mesh area** (`MESH_MIN`/`MESH_MAX` or the configured area).
2. If that rectangle is smaller than the minimum area size (`min_bedmesh_area_size`, 100 × 100 mm by default), it is enlarged around its centre up to that size, then limited to the mesh area again.
3. The number of points on each axis is the normal count multiplied by (new size ÷ normal size), **rounded up**, then the algorithm rules above are applied. A 9 × 9 mesh over a 100 × 100 mm part on a 400 × 400 mm area becomes roughly 3 × 3 to 4 × 4.
4. The mesh is measured, activated and stored in the profile named by `PROFILE` — which is `default` unless you say otherwise. **A partial mesh therefore replaces the full-bed `default` mesh** (for the current session). Keeping the full-bed mesh by storing the partial one under another name is risky (see the warning at the start of this chapter); `adaptive` is the only name seen to survive restarts.

*When it silently does nothing special.* If the four lines are missing (other slicer, non-print use, or a print that has ended — the information is erased when a print finishes or is cancelled), the whole bed is measured as if `ADAPTIVE=0`, and **no message tells you so**. The only message you can get is "virtualSD objects not enabled. Using full mesh..." in the unusual case that the print-job module is not running.

*Is it used by a normal print?* **No.** The stock start code of the sliced files only runs `G9111`, which reloads the stored `default` mesh but never measures a new one, so adaptive meshing never happens on its own. The only place in the firmware that runs `BED_MESH_CALIBRATE ADAPTIVE=1` is the touchscreen auto-leveling (`LEVIQ` → `LEVIQ_PROBE`); that is started with no print running, so it always measures the full bed (your printer's current `default` mesh is indeed a full 9 × 9 mesh from X/Y 10 to 410).

*How to make a print use it* (not tested on a printer): in the slicer's machine start G-code, **after** the `G9111` line, add

```
BED_MESH_CALIBRATE ADAPTIVE=1 ADAPTIVE_MARGIN=5 PROFILE=adaptive
```

The file must be printed from the printer's storage (started as a print job), not run by pasting its lines in the console, because the first-layer information is only collected while a print is running. Add the four comment lines yourself only if your slicer does not write them.

**Round beds only (not useful here):** `MESH_RADIUS`, `MESH_ORIGIN` and `ROUND_PROBE_COUNT` switch the grid to a circular pattern. The Kobra 3 Max has a rectangular bed; do not use them.

### Examples

```
G28
BED_MESH_CALIBRATE
```
Home, then measure the whole bed with the stock settings and store the result in the `default` profile.

> **Do not keep one mesh per bed surface or temperature under a name** (`PROFILE=pei_60C` and the like): see the warning at the start of this chapter. Measure again into `default` instead.

> `PROBE_COUNT`, `MESH_MIN` and `MESH_MAX` are ignored by this firmware (see the observation above), so no example is given for them.

```
BED_MESH_CALIBRATE ADAPTIVE=1 ADAPTIVE_MARGIN=5
```
In a print start code: measure only under the model (+ 5 mm border) of the file being printed.

### Notes and cautions

- The new mesh replaces the previous one immediately. If the measurement is interrupted (error, cancel), **no mesh is active** afterwards: load a profile with `BED_MESH_PROFILE LOAD=…`.
- The profile exists only until the next restart unless you run `SAVE_CONFIG` (it writes the saved values to disk; in this firmware it does **not** restart — the help text saying "and restart" is inherited from Klipper. On a Rinkhals-style printer the values go to `printer_mutable.cfg`, not `printer.cfg`. Moonraker's `configfile` object keeps showing the old values and the "pending" list until the printer program is restarted — see the `SAVE_CONFIG` note below. Still avoid it during a print unless you know why).
- A profile named `default` is what the stock pre-print sequence (`G9111`/`G9112`) reloads after its own operations.

---

## `BED_MESH_PROFILE` — Save, load or remove a profile

### What it does

> **⚠️ WARNING — do not use `SAVE` (or `REMOVE`) with a name other than `default`.** A saved named profile can lock the printer at its next start with **error 11407**. Details and the way out: see the warning at the start of this chapter. Only `BED_MESH_PROFILE LOAD=default` is safe to use.

Manages the named meshes kept in the printer's memory. You give **one** action per command, followed by a profile name:

- **`SAVE`** — stores the mesh currently in use under that name, so you can come back to it later. If a profile with the same name exists it is replaced. The profile is kept for the current session; run `SAVE_CONFIG` to make it permanent (it writes the file and does not restart; see the status-display note).
- **`LOAD`** — makes a stored profile the active mesh immediately (replacing the current mesh). This is instant: nothing is measured.
- **`REMOVE`** — deletes a stored profile. If it was saved to the configuration file, the deletion also becomes permanent only after `SAVE_CONFIG`.

### Syntax

```
BED_MESH_PROFILE SAVE=<name>
BED_MESH_PROFILE LOAD=<name>
BED_MESH_PROFILE REMOVE=<name>
```

### Parameters

Exactly one of the three is expected per call. If you give several at once, only one of them is carried out and which one is not defined — send them one per command.

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `SAVE` | Save the active mesh under this name. **Do not use** (error 11407 risk, see the warning above). | A name without spaces. **`default` is reserved and refused** for `SAVE` (`BED_MESH_CALIBRATE` creates the `default` profile itself). The bed must have been measured (a mesh must be active). | — (required) |
| `LOAD` | Activate the profile with this name. | Name of an existing profile (`default`, or one you saved). | — (required) |
| `REMOVE` | Delete the profile with this name. | Name of an existing profile. | — (required) |

### Messages you may see

| Message | Meaning |
|---|---|
| `Invalid syntax '…'` | None of `SAVE`/`LOAD`/`REMOVE` was given. |
| `Value for parameter 'SAVE' must be specified` | The action was given without a name. |
| `Profile 'default' is reserved, please choose another profile name.` | You tried `SAVE=default`. |
| `Unable to save to profile [name], the bed has not been probed` | No mesh is active: measure with `BED_MESH_CALIBRATE` or `LOAD` a profile first. |
| `bed_mesh: Unknown profile [name]` | `LOAD` of a profile that does not exist. |
| `No profile named [name] to remove` | `REMOVE` of a profile that does not exist. |
| `Bed Mesh state has been saved to profile [name] for the current session. The SAVE_CONFIG command will update the printer config file and restart the printer.` | `SAVE` worked (session only). |

### Examples

```
BED_MESH_PROFILE LOAD=default
```
Use the stored `default` mesh again without measuring (for example in your print start code).

```
BED_MESH_PROFILE REMOVE=<name>
SAVE_CONFIG
```
Only to get rid of a named profile that already exists, so that it can no longer lock the printer at the next start (not tested on a printer: check afterwards that `printer_mutable.cfg` no longer contains a `"bed_mesh <name>"` section, and do not restart the printer before that).

---

## `BED_MESH_CLEAR` — Switch the mesh off

### What it does

Deactivates the current mesh: from now on the printer applies **no** height correction. The stored profiles are not touched, so you can reactivate one with `BED_MESH_PROFILE LOAD=…`. The command also refreshes the printer's internal position so that the nozzle does not jump when the correction disappears.

The stock pre-print routines (`G9111`, `G9112`) use it to get a flat reference before leveling or wiping the nozzle.

### Syntax

```
BED_MESH_CLEAR
```

### Parameters

None.

### Example

```
BED_MESH_CLEAR     ; print without any bed compensation
```

---

## `BED_MESH_OUTPUT` — Show the mesh or the measurement points

### What it does

Writes information about the mesh in the console. It changes nothing.

- **Without parameter** it prints the active mesh: the measured heights as a table, the mesh size, its offsets, its average height, the smallest and largest height ("range") and the interpolation algorithm in use. If no mesh is active it answers `Bed has not been probed`.
- **With `PGP=1`** it instead prints the list of points that the next `BED_MESH_CALIBRATE` will measure (with their numbers — useful to choose `RELATIVE_REFERENCE_INDEX`), the reference point and any excluded "faulty regions".

### Syntax

```
BED_MESH_OUTPUT [PGP=0|1]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `PGP` | "Print generated points": show the planned measurement points instead of the current mesh. | `0` = show the current mesh. Any value above 0 (use `1`) = show the planned points. | `0` |

### Examples

```
BED_MESH_OUTPUT           ; table of heights of the active mesh
BED_MESH_OUTPUT PGP=1     ; numbered list of the points that would be measured
```

Reading the result: heights are in mm relative to the reference (a positive number means the bed is higher there). A "range" of a few tenths of a millimetre is normal; much more points to a bed or build-plate problem.

---

## `BED_MESH_MAP` — Mesh in machine-readable form

### What it does

Prints the active mesh as one line of text that starts with `mesh_map_output` followed by a data structure (JSON) containing the lowest and highest X/Y positions of the mesh (`mesh_min`, `mesh_max`) and the measured heights (`z_positions`). Web interfaces such as Mainsail use it to draw the colour heat-map of the bed; you normally do not need to type it. If no mesh is active it answers `Bed has not been probed`.

### Syntax

```
BED_MESH_MAP
```

### Parameters

None.

---

## `BED_MESH_OFFSET` — Shift the mesh in X/Y (**not working**)

### What it should do

Shift the whole mesh by an offset in X and Y (useful with probes that are not at the nozzle position).

### What it really does in firmware 2.5.2.8

**Nothing useful — do not use it.** When no mesh is loaded it answers `No mesh loaded to offset`. When a mesh is loaded the command **fails with an internal error** and the mesh is not changed (the program tries to store the offsets in a list that was never created, and it also looks for its parameters in lower case, so they are never found). This is a defect of the firmware, not something you can work around with different parameters.

### Syntax (for reference)

```
BED_MESH_OFFSET X=<mm> Y=<mm>
```

### If you need to shift the print relative to the mesh

Change the stored probe offset in the configuration or use `SET_GCODE_OFFSET` (covered in the G-code state chapter) instead.

---

### `SAVE_CONFIG`: what it does and what the status shows (observed on a printer)

- `SAVE_CONFIG` writes the staged values to disk and does **not** restart the firmware. On a Rinkhals-style printer they are written to `printer_mutable.cfg` (visible through Moonraker at `/server/files/config/printer_mutable.cfg`), not to `printer.cfg`.
- **Observed:** after a `SAVE_CONFIG` the file held the new value (`probe` `z_offset` −0.0150), but Moonraker's `configfile` object still showed the old value in `settings` (−0.1875) and still listed the item in `save_config_pending_items`, with `save_config_pending` = True. So that status is **not** a reliable "unsaved" indicator, and the old values in `settings` may not reflect what is in effect. To know what is really saved, read the file; to know what is live, read the object concerned (for a Z offset, `gcode_move.homing_origin`).
- Whether the firmware re-reads the saved file without a restart was not determined; the earlier statement in this guide that it "reloads in place" was an assumption from reading the code and is not confirmed by the status display. **Observed after a `RESTART`:** `configfile.settings` showed the value from the saved file (`probe` `z_offset` −0.1050 in `printer_mutable.cfg`), `save_config_pending` was False and the pending list empty. So a restart does bring the status in line with the file; whether the firmware also applies a saved file *without* a restart is still not confirmed.

*End of chapter 3. Next chapter in the command table: **Calibration** (`PA_AUTO_CALIBRATE`, `PID_CALIBRATE`, `TUNING_TOWER`).*


---

# Chapter 4 — Calibration

Three commands that measure or tune something and then change a setting for you.

| Command | In one line |
|---|---|
| `PA_AUTO_CALIBRATE` | Automatically finds the **pressure advance** value (`K`) of the loaded filament, using the nozzle's pressure sensor, and applies it for the current session. Temperatures come from the last analysed G-code file first. |
| `PID_CALIBRATE` | Tunes the **temperature controller** (PID) of the nozzle or the bed. The result is kept only after `SAVE_CONFIG`. |
| `TUNING_TOWER` | While you print a "tower" test model, changes one setting step by step as the height increases, so you can pick the best value by eye. |

---

## `PA_AUTO_CALIBRATE` — Find the pressure advance automatically

> **Warning — the result overrides your `M900 K` in the next print.** After a successful run (stock setting `enable_dyna_PA: false`), the firmware replaces the `K` of **every** `M900 K…` / `SET_PRESSURE_ADVANCE ADVANCE=…` by the calibrated value, including the one in your slicer's start G-code. The calibration is kept in memory and is only dropped at the **end or cancel of the next print** (which then also sets `K` back to 0.02), at a firmware restart, or by a failed re-calibration. So a `K` test made after a `PA_AUTO_CALIBRATE` (for example a pressure-advance tower) shows no change at all. **To check which `K` is active:** read `pressure_advance` in the `extruder` object (Mainsail / Moonraker status) after your start G-code has run; it shows the value really applied (the calibrated one if the override is active). Nothing is printed in the console by `M900`. In dynamic mode (`enable_dyna_PA: true`) the `K` used for each move is computed per move and cannot be read.

*Pressure advance (`K`)* compensates for the elasticity of the plastic in the nozzle: the extruder pushes slightly more filament while accelerating and pulls back while braking, which sharpens corners. The right value depends on the filament. This command measures it for you with the pressure sensor in the nozzle (the same kind of sensor used for bed leveling).

### Where the temperatures and the speed come from

Before doing anything, the command looks at the **last G-code file the printer analysed** (the printer reads the comment lines of a file's header when a print starts) and uses these lines if they are present:

| Setting for the test | Taken from | Rule |
|---|---|---|
| Nozzle temperature | `; nozzle_temperature =` in the file | If the file gives **200 °C or more**, it is used and `EXTRUDERTEMP` is **ignored**. Otherwise `EXTRUDERTEMP` (default 220) is used. In both cases the result is forced into **200–250 °C**. |
| Bed temperature | `; first_layer_bed_temperature =` in the file | If the file gives a value above 0, it is used and `BEDTEMP` is **ignored**. Otherwise `BEDTEMP` (default 60). |
| Extrusion speed of the test | larger of `; outer_wall_speed =` and `; perimeter_speed =` in the file | Forced into **90–400 mm/s** (90 mm/s if the file has none). |

So after you have printed (or started) a file from the slicer, the command will quietly reuse that file's temperatures. To be sure of the values, check the first console lines after the start, or start a print of a file without those comments first. Whether these remembered values are cleared when a print ends was not determined **(to verify)**; right after a firmware restart they are empty and your parameters apply.

### What it does

1. **Allows the nozzle to be cleaned again** (`G9113`) and switches the part-cooling fan off.
2. **Starts heating** the bed (`M140`) and nozzle (`M104`) to the temperatures above, without waiting.
3. **Homes if needed** (`G28 W` when X, Y and Z are not all homed) and lifts the nozzle by a safety distance.
4. **Moves to the calibration position** (9600 mm/min), waits for the end of the move, then **waits for the nozzle temperature** (`M109 … I1`).
5. **Purges 10 mm of filament** (`G92 E0`, `G1 E10 F1000`).
6. **Measures.** The pressure sensor is switched on and the printer tries a series of `K` values: starting at **0.015** and going up in steps of **0.005** until about **0.075**. For each value it sets `M900 K<value>` (relative extrusion `M83`), extrudes **1 mm** then **2 mm** at the test speed, and reads the pressure; a curve fitted to the readings points to the best `K`. These limits are fixed in the firmware: the `pa_min` / `pa_max` entries of `[flow_calibration]` are not used by this routine.
7. **Applies the result.**
   - **Normal case (the stock setting, `enable_dyna_PA: false`, "static" calibration):** the first (best) value is rounded to 3 decimals, stored for the current filament and applied with `M900 K<value>`. The log shows `PA: Static ,filamentIndex=… best_k = …`.
   - **If `enable_dyna_PA: true` is set in `printer.cfg` ("dynamic" calibration):** three measurements at three speeds are made and stored, and `K` is then adjusted per move during printing. See "Static and dynamic calibration in detail" below.
8. **If the measurement fails** (for example the sensor gives no usable data), the log shows an error, **no `M900 K` is sent for the result**, and the calibration is flagged as failed for that filament. The test values set by step 6 may still be active until the print ends **(to verify)**.

With `USE_ACE=1` the preparation is different: the filament hub is used to wipe/sweep the nozzle (`G9112`, fan 100 %, heating with `M109 … I1`) and `G9113` is run again at the end.

### Syntax

```
PA_AUTO_CALIBRATE [EXTRUDERTEMP=<°C>] [BEDTEMP=<°C>] [USE_ACE=0|1]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `EXTRUDERTEMP` | Nozzle temperature for the test — **only used if the last analysed G-code file does not give 200 °C or more** (see table above). | Number; below 200 becomes 200, above 250 becomes 250. | 220 |
| `BEDTEMP` | Bed temperature for the test — **only used if the last analysed file gives no first-layer bed temperature.** | Number in °C. | 60 |
| `USE_ACE` | Use the ACE filament hub for the preparation. | `0` = no, other value = yes | 0 |

Write parameters as `NAME=value` without spaces around `=`.

### Examples

```
PA_AUTO_CALIBRATE EXTRUDERTEMP=230 BEDTEMP=70
```
Calibrates at 230 °C / 70 °C (provided no file with its own temperatures was analysed).

```
PA_AUTO_CALIBRATE
```
Uses the defaults 220 °C / 60 °C (or the temperatures from the last file).

### Notes and cautions

- Load the filament first and make sure the nozzle is clean: the test extrudes plastic. Keep the calibration position free.
- **The result is not saved to the configuration.** It is kept in memory only, until a firmware restart or the end/cancel of the next print. Read the `best_k` value in the log if you want to keep it (for example for `printer.cfg`).
- While a calibration is valid (static mode), the `M900 K…` lines of your G-code are ignored; see "Static mode" below. The calibration is cleared at the end or cancel of every print and `K` is then set to 0.02.
- Nozzle temperatures outside 200–250 °C cannot be used with this command, whatever you type.
- Verified in the firmware code: the sources of temperature/speed, the sweep (0.015 → 0.075 in 0.005 steps, 1 mm + 2 mm extrusions), the rounding to 3 decimals, and that nothing is applied on failure. Not verified: the mathematical fit itself, the number of repeats, and what happens to the temporary `K` after a failure.

### Using it in practice (static mode, stock settings)

Pressure advance on this printer starts from **0.02**: `pressure_advance: 0.02` in the `[extruder]` section of `printer.cfg` gives that value at every firmware start (the smoothing time is not set in the file; the firmware default is 0.04 s). The firmware also sets `K` back to 0.02 (`M900 K0.02`) at the end or cancel of every print. So 0.02 is the "normal" value whenever no calibration is active.

There are two ways to work, depending on whether a calibration is active when the print starts.

#### Scenario A — you calibrate before the print

1. Load the filament, heat-soak as usual, then run `PA_AUTO_CALIBRATE` (see the parameters above; with a G-code file already analysed, its temperatures are used). Wait for it to finish; the log shows `PA: Static ,filamentIndex=… best_k = …`.
2. Start your print **without doing anything else that cancels it** (do not cancel or finish another print in between, do not restart the firmware).
3. The printer uses the calibrated `K` for the whole print. **Every `M900 K…` and `SET_PRESSURE_ADVANCE ADVANCE=…` in your G-code is ignored** (only the smoothing time is still read), so you do not need to remove or change them in the slicer.
4. When the print ends or is cancelled, the calibration is erased and `K` goes back to 0.02.

The calibrated value is therefore used for **one print only**. For the next print you calibrate again (or follow scenario B).

#### Scenario B — no calibration just before the print

1. The calibration tables are empty (cleared at the end of the previous print, or never filled since the last restart).
2. `M900` and `SET_PRESSURE_ADVANCE` work normally: the `K` in your slicer's start G-code (for example `M900 K0.035`) is applied.
3. If your G-code contains no `M900` / `SET_PRESSURE_ADVANCE`, `K` stays at **0.02** (stock value at startup and after every print).

To use a `K` you have found by calibration in this scenario, read `best_k` in the log (or the console) and put it in your slicer's start G-code, or in `printer.cfg` under `[extruder]` (`pressure_advance: …`), which then becomes the value at every startup.

#### Things to be careful with

- **Changing filament:** the printer does not know which filament the calibration was done for (without the ACE hub it keeps one value for "whatever is loaded"). If you calibrate PLA and then load PETG without printing in between, the PLA value is still forced. Calibrate again after a filament change.
- **A failed calibration** does nothing harmful: it is recorded as failed and your own `M900` values work again.
- **A calibration is lost** by a firmware restart, by the end or the cancel of a print.
- The ACE hub and the dynamic mode work differently; see the next section.

### Static and dynamic calibration in detail

The behaviour depends on one switch in `printer.cfg`, section `[flow_calibration]`: `enable_dyna_PA`. The stock value is `false` (**static** calibration). Changing it needs a firmware restart. Everything below was read from the firmware code and has not been run on a printer.

#### Static mode (`enable_dyna_PA: false`, the stock setting)

**What the calibration does.** One measurement pass at the test speed (see the table above). The best `K` is rounded to 3 decimals, stored, and applied with `M900 K<value>`.

**Where the result is stored.** In the printer's memory only (RAM), in three small tables kept by the pressure-advance module. Each table is indexed by a **filament number**:

| Table | Content for the filament number |
|---|---|
| Status | `1` = calibration succeeded, `2` = calibration failed (the entry does not exist before any calibration) |
| Static `K` | the rounded best `K` |
| Dynamic table | the three speeds and their `K` (dynamic mode only) |

Nothing is written to `printer.cfg` or to any file, so the result is lost at a firmware restart or reboot. The filament number is the number of the filament slot when the ACE hub is used. Without the hub it is a fixed number (0 after a restart, **100** after the first reset described below), so the calibration applies to "whatever filament is loaded"; if you change filament you must run `PA_AUTO_CALIBRATE` again.

**How the printer checks that a calibration was done.** Each time it receives `M900` (or `SET_PRESSURE_ADVANCE`) it looks at the status table for the current filament number. The calibration counts as valid only if **the status is `1` and dynamic mode is off**. In that case:

- the `K` (`M900`) or `ADVANCE` (`SET_PRESSURE_ADVANCE`) written in the command is **ignored** and the stored static `K` is used instead, limited to 0 … 0.2;
- the smoothing time of the command (`S` for `M900`, `SMOOTH_TIME` for `SET_PRESSURE_ADVANCE`) is still read as usual.

If there is no status, or it is `2` (a failed calibration), `M900` / `SET_PRESSURE_ADVANCE` work normally and use the value you give.

**`SET_PRESSURE_ADVANCE` works exactly like `M900`.** Their code is the same: both look up the calibration module, check the status of the current filament number, and in static mode replace the requested value by the stored `K` (same 0 … 0.2 limit). Only the parameter names differ (`M900 K… S…` against `SET_PRESSURE_ADVANCE ADVANCE=… SMOOTH_TIME=…`). Both end up in the same function that sets the pressure advance of the extruder.

**When the calibration is cleared.** At the **end of every print and when a print is cancelled**, the firmware empties all three tables, sets the filament number to 100 and switches dynamic mode **off** (the stored value of `enable_dyna_PA` is not restored until a restart; whether something switches it on again at the next print start was not determined **(to verify)**). Right after that it sends `M900 K0.02`, which therefore sets `K` to 0.02 (the same value as `pressure_advance` in the stock `[extruder]` section, used at startup) as no calibration exists any more. A calibration made while idle thus lasts until the end of the next print; one made for a print (for example by the ACE hub at print start) lasts for that print. The tables are also cleared when a G-code file whose header declares `test_mode 2` is started (an unsupported mode; the log says "AC Slicer test_mode 2 is not support!").

To drop a calibration by hand, restart the firmware, finish or cancel a print, or run `PA_AUTO_CALIBRATE` again (a failed run sets the status to `2`, which gives `M900` back to you).

#### Dynamic mode (`enable_dyna_PA: true`)

Idea: the best `K` depends on the printing speed (faster extrusion = more pressure). Instead of one `K`, the printer measures three and then adjusts `K` continuously during the print according to the speed of each move.

**1. Calibration (`PA_AUTO_CALIBRATE`)** — same preparation as above, but:

- The measurement is run **three times, at 300, 200 and 100 mm/s** (18 000, 12 000 and 6 000 mm/min). The speed taken from the G-code file is not used. The temperature rules (file first, 200–250 °C) are unchanged.
- For each speed the same `K` sweep is done (0.015 → 0.075, step 0.005; 1 mm then 2 mm of extrusion per value). The fitted result is corrected by a factor read from the `interp_fit_factor` table of `[flow_calibration]` (4 speeds × 5 temperatures: 100–400 mm/s, 210–250 °C, interpolated; speed and temperature are limited to those ranges). The tuning entries `delta_press_max` and `fit_val_max` limit the pressure readings accepted by the fit.
- The three best values (lowest speed first) are then checked:
  - **Normal result** — the three values do not rise steadily with speed. They are made consistent so that `K` never increases with speed: if the 100 mm/s value is above the 200 mm/s value, the 300 mm/s value is capped at the 200 mm/s value; if the 200 mm/s value is above both others, the 100 mm/s value is raised to it. The log says `PA: Dynamic Calibration Successful`, then `PA: Dynamic Calibration, filamentIndex=… best_k=…`. The command sends **no** `M900` in this case.
  - **Suspect result** — the values rise strictly with speed (low < middle < high). The calibration is marked as failed. If the highest and lowest differ by more than 0.002, the log shows `PA: Error, can not get Dynamic PA value.` and nothing else happens. If they differ by 0.002 or less, the log shows `PA: MinBestK … MedianBestK … MaxBestK …` and `PA: Unstable Dynamic Calibration results, Use MedianBestK as Static Calibration.`, and the **median** is applied once with `M900 K<median>`.
- The table (three speeds and their `K`) and the status are stored for the current filament number, in memory only, and cleared at the end or cancel of every print (see above, which also switches the dynamic mode off).

**2. During the print** — every extruder move asks the calibration for a `K`:

- If the calibration of the current filament was successful, `K` is found by **linear interpolation** between the three calibrated points for the speed of the move, then **limited to the range `pa_min` … `pa_max`** (stock: 0.015 … 0.075). What happens for speeds outside 100–300 mm/s (clamped or extrapolated) was not determined **(to verify)**.
- If there is no valid calibration, the dynamic part returns nothing and the normal pressure advance (`M900 K`, `SET_PRESSURE_ADVANCE`) applies. Whether `M900` still has an effect while the dynamic value is in use was not determined **(to verify)**.

**3. With the ACE filament hub** — when dynamic mode is on, the hub itself starts the calibration: at the start of a print and at each filament change (`T` command) it checks whether the filament in use already has a calibration and, if not, runs `PA_AUTO_CALIBRATE USE_ACE=1` after setting the fan. The values are stored **per filament slot**. (The exact test "already calibrated or not" was not decoded **(to verify)**.) Without the hub nothing runs by itself: you start `PA_AUTO_CALIBRATE` yourself, once per filament.

**Summary of the settings** (all in `[flow_calibration]` of `printer.cfg`; stock values in brackets): `enable_dyna_PA` (false), `pa_min` (0.015) and `pa_max` (0.075) — limits of the `K` used during dynamic printing, `pa_min_nozzled_comp` (0.01), `delta_press_max` (50 000), `fit_val_max` (10 000), `interp_fit_factor` (20-value table), `extru_load`. Note that `pa_min` / `pa_max` limit the dynamic `K` while printing but do **not** limit the sweep in the calibration itself, which is fixed.

---

## `PID_CALIBRATE` — Tune the heater controller

A heater holds its temperature with a PID controller (three numbers: `Kp`, `Ki`, `Kd`). If the temperature oscillates or overshoots (new heater, new sensor, different fan setup), run this test.

### What it does

1. **Homes if needed** (`G28 W`) — only when `homing_first: 1` in the `[pid_calibrate]` section (stock: yes) — then switches to absolute positioning.
2. **Moves to a test position** (stock: X 457, Y 220, from `move_xy_position`; speed 6000 mm/min) and lifts Z to 15 mm (`move_z_up`).
3. **Takes over the chosen heater** and heats it to `TARGET`, letting the temperature swing above and below it several times while recording the oscillation. The test stops with an error if the heater name is unknown (`Lookup_heater … failed`) or the test is interrupted (`pid_calibrate … interrupted`).
4. **Computes `Kp`, `Ki`, `Kd`**, writes them to the log (`PID parameters: pid_Kp=… pid_Ki=… pid_Kd=…`) and **stages them in the configuration** (`control: pid` in the heater's section).
5. If the heater was the **nozzle**, a cooling routine follows: the nozzle is moved to the filament-hub sweep position and the part-cooling fan runs at 100 % for a while, then off **(to verify)**.
6. If `WRITE_FILE=1`, the recorded temperature curve is also written to `/tmp/heattest.txt` on the printer.

The new values become permanent only after **`SAVE_CONFIG`** (which writes the file; no restart in this firmware).

### Syntax

```
PID_CALIBRATE HEATER=<extruder|heater_bed> TARGET=<°C> [WRITE_FILE=0|1]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `HEATER` | Which heater to tune. | `extruder` or `heater_bed` | none — **required** |
| `TARGET` | Temperature around which the test oscillates. Use the temperature you usually print at. | °C, within the heater's limit (stock max: nozzle 320, bed 110) | none — **required in practice** (the firmware reads 0 if you omit it, which makes no useful test) |
| `WRITE_FILE` | Also save the recorded curve to `/tmp/heattest.txt`. | `0` or `1` | 0 |

### Examples

```
PID_CALIBRATE HEATER=extruder TARGET=220
SAVE_CONFIG
```
Tunes the nozzle at 220 °C, then keeps the result (the firmware restarts).

```
PID_CALIBRATE HEATER=heater_bed TARGET=60
```
Tunes the bed at 60 °C.

### Notes and cautions

- Stock values for reference: nozzle `Kp 19.792 / Ki 1.807 / Kd 54.179`; bed `Kp 53.444 / Ki 0.563 / Kd 1268.631`.
- The heater runs hot and unattended: stay near the printer and do not run it during a print.
- Test with the part-cooling fan setting you normally print with, in a room without a draught. The bed test takes several minutes.
- Until you run `SAVE_CONFIG` the old values are still the ones in `printer.cfg`; they are used again after a restart.

---

## `TUNING_TOWER` — Change a setting with the height

Print a tall test object (a "tower") and let the printer change one setting every few millimetres. Afterwards you look at the tower, note at which height it looks best, and compute the matching value. Typical uses: pressure advance, temperature, speed, acceleration.

### What it does

1. **Ends any tuning test already running** (message `Ending tuning test mode`).
2. **Reads the parameters** and starts the new test (message `Starting tuning test (…)` with the values).
3. From then on, **every move is inspected**: the value for the current height is computed and, whenever it changes, `COMMAND` is sent with the new value. It is sent as `<COMMAND> <PARAMETER>=<value>`; for classic commands such as `M104` or `M900` it is sent as `<COMMAND> <PARAMETER><value>` (e.g. `M900 K0.040`).
4. The value is:
   - **linear** (with `FACTOR`): `START + FACTOR × height`;
   - **stepped** (with `STEP_DELTA` and `STEP_HEIGHT`): `START + STEP_DELTA × (number of whole steps of STEP_HEIGHT below the current height)`;
   - if `BAND` is set, the height is first rounded to the middle of a band of that thickness, so the value stays constant inside each band;
   - if `SKIP` is set, that much height at the start is ignored (the value stays at `START`).

(The formulas are those of the standard Klipper command, which this one reproduces; not checked on a printer **(to verify)**.)

### Syntax

```
TUNING_TOWER COMMAND=<command> PARAMETER=<name> START=<value> FACTOR=<per mm> [BAND=<mm>] [SKIP=<mm>]
TUNING_TOWER COMMAND=<command> PARAMETER=<name> START=<value> STEP_DELTA=<value> STEP_HEIGHT=<mm> [SKIP=<mm>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `COMMAND` | The command that changes the setting. | e.g. `SET_PRESSURE_ADVANCE`, `M900`, `M104`, `SET_VELOCITY_LIMIT` | none — **required** |
| `PARAMETER` | Name of that command's parameter to vary. | e.g. `ADVANCE`, `K`, `S`, `ACCEL` | none — **required** |
| `START` | Value at the bottom of the tower. | number | 0 |
| `FACTOR` | Change of the value per mm of height. | number (may be negative) | 0 |
| `STEP_DELTA` | Change of the value at each step (used with `STEP_HEIGHT`). | number (may be negative) | 0 |
| `STEP_HEIGHT` | Height of one step. | mm | 0 |
| `BAND` | Round the height into bands of this thickness. | mm | 0 (off) |
| `SKIP` | Height at the start that stays at `START`. | mm | 0 |

### Examples

```
TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=.005
```
Pressure advance rises by 0.005 per mm: 0 at the bottom, 0.1 at 20 mm. If the best-looking part is at 8 mm, the best value is 8 × 0.005 = 0.04.

```
TUNING_TOWER COMMAND=M104 PARAMETER=S START=250 STEP_DELTA=-5 STEP_HEIGHT=10
```
Temperature tower: 250 °C at the bottom, 5 °C lower every 10 mm.

### Notes and cautions

- Send it **after the first layer has started** (or from the slicer's start G-code), before the tower grows. Send `TUNING_TOWER` again (any values) or restart the firmware to end the mode afterwards.
- Use only one of the two modes (`FACTOR`, or `STEP_DELTA` + `STEP_HEIGHT`). The firmware gives no error for missing or contradictory values.
- With `M104`, check that the range stays inside your filament's limits and the printer's (on the stock configuration `M104`/`M109` silently cap the nozzle at 300 °C — see chapter 7).

---

*End of chapter 4.*

---

# Chapter 5 — Core G-code

Nine basic commands: talking to the console, asking the printer about itself, restarting, and the emergency stop. Several of them are standard Klipper commands, but **two behave very differently from what you may expect: `M117` and `M110`** (see below).

| Command | In one line |
|---|---|
| `ECHO` | Writes the line to the firmware log (**not** to the console). |
| `FIRMWARE_RESTART` | Stops the printer software and asks for a full restart. |
| `HELP` | Lists the commands that have a help text. |
| `M110` | Does nothing (accepted only so that slicers/hosts do not get an error). |
| `M115` | Reports the firmware name and version. |
| `M117` | **Does not show a message.** At print start it switches the X/Y motors to the "print" driver mode and resets the print timer. |
| `M8844` | Emergency stop, identical in effect to `M112`, with an optional reason text. |
| `RESTART` | Stops the printer software and asks for a restart (reloads the configuration). |
| `STATUS` | Tells whether the printer is ready. |

---

## `ECHO` — Write a line to the firmware log

### What it does

1. Takes the **whole command line exactly as it was typed** (including the word `ECHO`) and passes it to the "info" responder **with its console flag off**.
2. With that flag off the responder **only writes the line to the firmware log** (on a Rinkhals printer `/tmp/gklib.log`). **It is not sent to the Mainsail console**, whether typed in the console or run from a print file. *(Read from the disassembly of `Cmd_ECHO` and `Respond_info`, 2026-11-13; a user test confirmed that `M115` from a print file shows in the console and `ECHO` does not. An earlier version of this guide said the opposite: that was wrong.)*

To print your own text in the console, a macro with `action_respond_info` is needed (that action calls the responder with the console flag on); see `ORCA_SLICER_GCODE.md`.

It has no parameters and changes nothing in the printer. The only use is to leave a trace in the log (for example to mark a step of a start code when you read `gklib.log`).

### Syntax

```
ECHO [any text]
```

### Examples

```
ECHO hello
```
Nothing appears in the console; the log gets a line containing `ECHO hello`.

### Notes and cautions

- The logged text is the whole line, including the word `ECHO`.

---

## `HELP` — List the commands that have a help text

### What it does

1. Builds a list of all registered commands **that have a help text**, sorted alphabetically.
2. Sends it to the console, one line per command, in the form `NAME      : help text` (name padded to 10 characters).
3. If the printer is **not ready** (for example after an error), the first line says *"Printer is not ready - not all commands available."*

### Syntax

```
HELP
```

### What you will see

On this printer **only a few commands have a help text** (the live printer's list of 102 commands is mostly the standard Klipper ones). Most Anycubic-specific commands (`G9111`, `LEVIQ*`, `M600`…) have none, so **they do not appear in `HELP` even though they exist**. Do not use `HELP` to decide whether a command is available; use this guide or `curl http://<printer>:7125/printer/gcode/help`.

### Notes and cautions

- The four commands of this chapter that have help texts are `HELP`, `STATUS`, `RESTART` and `FIRMWARE_RESTART`; `ECHO`, `M110`, `M115`, `M117`, `M8844` have none.

---

## `M110` — Set line number (ignored)

### What it does

Nothing. The command is accepted and only writes a line to the debug log. The line number given (`N…`) is **not** used for anything, and no error is raised.

It exists so that hosts/slicers that send `N123 … *checksum` / `M110 N0` lines work without errors.

### Syntax

```
M110 [N<line>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `N` | Line number (ignored). | Any | — |

---

## `M115` — Report firmware name and version

### What it does

1. Builds the text `FIRMWARE_NAME:project FIRMWARE_VERSION:<version> ` — the firmware's internal name is literally `project`, and the version is the software version of the printer program.
2. Sends it to the console (as an `ok …` answer when the command comes from a host that expects acknowledgements, otherwise as an info line).

### Syntax

```
M115
```

### Notes and cautions

- The **order of the two items is not fixed** (it can change from one call to another).
- The version string may be **empty**: on the printer used for the analysis (running a customised configuration `rinkhals_gklib.cfg`) the software version reported by the API is empty. This is **not** the firmware version of the Anycubic package (2.5.2.8).
- No other Marlin capabilities (`Cap:…`) are reported.

---

## `M117` — Display message… but in practice "print start" signal

In normal Marlin/Klipper, `M117 text` displays a message on the LCD. **In this firmware the text is completely ignored.** Instead, the command sends an internal "**start of print**" signal. The slicer start G-code of Anycubic Slicer Next contains `M117` for exactly this reason.

### What it does

1. Marks the print job as **started** (internal flag of the print-file reader).
2. **Switches the stepper drivers of the X and Y motors from the quiet "StealthChop" mode to the normal "SpreadCycle" mode** — the same as sending `CLOSE_STEALTHCHOP STEPPER=stepper_x` and `…stepper_y`. On the Kobra 3 **Max** the second Y motor is included (`stepper_y1`), i.e. three motors in total. (The Kobra 3 and Kobra 3 V2 models only get X and Y; the model is recognised from the configuration's `device_type`, which is `Anycubic Kobra 3 Max` on this printer. **To verify**: the model test was read from the machine code and not tested.)
3. **Waits 0.1 s.**
4. Resets the **print-duration counter** (the "initial duration" of the print statistics), so that the time spent on preparation (heating, leveling) is not counted in the print time shown.

### Syntax

```
M117 [anything]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| (text) | Ignored. | — | — |

### Examples

```
M117
```
Normal use, at the end of the start G-code, just before the first layer.

```
M117 Hello
```
Does **not** show "Hello" anywhere; it still does everything above.

### Notes and cautions

- **Do not use `M117` as a message command in macros or G-code.** Each call switches the motor drivers to SpreadCycle and resets the print timer. Mid-print it will therefore **reset the elapsed-time display** (to verify) and make the motors noisier.
- To check the effect, query the drivers afterwards with `DUMP_TMC` (chapter 26).
- The print-timer reset (point 4) only happens if the print-statistics module is present, which is the case in the stock configuration.

---

## `M8844` — Emergency stop with a reason

### What it does

1. Looks for `S=` in the line. If followed by text, that text becomes the **reason**; if there is no `S=` (or nothing after it), the reason is the standard *"Shutdown due to M112 command"*.
   - `S="some text"` or `S='some text'` — the text between the quotes is used.
   - `S=some text` — everything after `S=` up to the end of the line is used.
2. Puts the printer in **shutdown**: all heaters and motors are stopped, the reason is shown as an error in the console, and **the printer refuses further commands until you send `FIRMWARE_RESTART`** (or restart from the screen / Mainsail).

It is the same mechanism as `M112`; `M8844` only adds a custom reason.
### Syntax

```
M8844 [S="<reason>"]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `S=` | Text shown as the reason for the shutdown. Must be written **with the equals sign** and, if it contains spaces, in quotes. | Free text | *Shutdown due to M112 command* |

### Examples

```
M8844
```
Immediate emergency stop.

```
M8844 S="Nozzle clogged"
```
Emergency stop; the console shows "Nozzle clogged" as the reason.

### Notes and cautions

- **A print in progress cannot be resumed** after this: the job is lost. Use `PAUSE` (chapter 20) if you only want to stop temporarily.
- The text is taken from the raw line; a text containing quotes of the other kind (`S="it's"`) works, but a closing quote is optional.

---

## `RESTART` — Restart the printer software

### What it does

1. If the printer is ready: informs the other modules that a restart is coming, **waits 0.5 s, then waits until all queued moves are finished**.
2. Asks the system to restart the printer program (`gklib`) and **reload `printer.cfg`**. Nothing is saved, and the command itself does not switch the heaters off (what happens to them while the program is down was not analysed **(to verify)**).
3. If the printer is not ready (error state), the restart request goes out immediately.

### Syntax

```
RESTART
```

### Notes and cautions

- A print in progress is **lost**. Use it only when idle or to recover from an error.
- Nothing is restarted *by the command itself*: the program just exits and relies on the system (service supervisor) to start it again. On this printer the live configuration is a Rinkhals one, where that restart is done by the surrounding system; with the stock Anycubic system the same is expected **(to verify, not tested)**.
- **Rinkhals blocks this command.** On a printer running Rinkhals, Moonraker (its `kobra.py` component) refuses `RESTART` and `FIRMWARE_RESTART` before they reach the printer program and prints: "RESTART / FIRMWARE_RESTART is not supported on GoKlipper. The command deadlocks gklib until the printer is power-cycled" (the issue number it quotes could not be matched to a Rinkhals issue about this). It refuses both the console and the HTTP routes (`/printer/restart`, `/printer/gcode/script`). To apply configuration changes, reboot the printer from the touch panel or power-cycle it.
- **Tested on a printer (idle, cold), bypassing Moonraker** by sending `RESTART` straight to the printer program's socket: a single `RESTART` did **not** deadlock the printer. Sending it **twice**, however, left `gklib` and Moonraker unresponsive. So Rinkhals' warning appears justified: avoid `RESTART` and `FIRMWARE_RESTART` on this firmware, and above all never send them repeatedly. Do not attempt it during a print.
- **After a restart the live G-code Z offset is gone.** Observed: the live offset (`gcode_move.homing_origin`, Z) was non-zero before the restart and 0 after it. Only what was saved to the configuration file survives (for the probe offset: `probe` `z_offset`). Baby-step adjustments and `SET_GCODE_OFFSET` values must be re-applied, or saved with `Z_OFFSET_APPLY_PROBE` + `SAVE_CONFIG`.
- Mainsail shows the connection as lost for a few seconds.

---

## `FIRMWARE_RESTART` — Restart the printer software (full)

### What it does

**Same Rinkhals restriction and test result as `RESTART` (see above): blocked by Rinkhals' Moonraker; a single direct send worked, a second one made `gklib` and Moonraker unresponsive.**

Identical to `RESTART`, except that it requests a **"firmware restart"**: the printer program exits with the request code `firmware_restart` instead of `restart`. On a classic Klipper this also resets the microcontrollers; **whether this printer's system treats the two codes differently is not known** **(to verify)**. It is the command that clears a shutdown caused by an error or `M8844`/`M112`.

### Syntax

```
FIRMWARE_RESTART
```

### Notes and cautions

- Same cautions as `RESTART`: a print in progress is lost.

---

## `STATUS` — Is the printer ready?

### What it does

1. If the printer is ready: answers `project state: Ready` in the console.
2. If it is not ready (starting, in error, in shutdown): the command **fails with an error** that contains the reason and the line `state: Not ready`.

### Syntax

```
STATUS
```

### Notes and cautions

- The word `project` in the answer is the firmware's internal name (see `M115`).
- Handy in a macro: the macro stops with an error if the printer is not ready.

---

*End of chapter 5.*

---

# Chapter 6 — Exclude object (cancel one part during a print)

When several parts are printed on the same plate, you can **cancel a single object** while the print goes on (the "cancel object" button of Mainsail / the touchscreen). This needs the G-code file to be labelled by the slicer. Six commands and one stock macro are involved.

| Command | In one line |
|---|---|
| `EXCLUDE_OBJECT_DEFINE` | Declares an object (name, centre, outline) — written by the slicer at the top of the file. |
| `EXCLUDE_OBJECT_START` | Marks "from here on, the G-code belongs to object X". |
| `EXCLUDE_OBJECT_END` | Marks the end of that object's G-code. |
| `EXCLUDE_OBJECT_END_NO_OBJ` | Ends the current object without naming it (for purge towers etc.). |
| `EXCLUDE_OBJECT` | Cancels (or restores) an object, or lists the cancelled ones. |
| `EXCLUDE_OBJECT_SET_OBJ` | Anycubic addition: declares *N* anonymous objects named `0`, `1`, … (used by `M486 T`). |
| `M486` (macro in `printer.cfg`) | Marlin/OctoPrint-style front end for the commands above. |

## How it works in this firmware (read this first)

The firmware does **not** do it the way standard Klipper does (no rewriting of moves). Instead:

1. **At print start**, the printer reads the whole file in the background. Every `EXCLUDE_OBJECT_DEFINE` line is executed immediately, so the list of objects (with their outlines) is available to the screen/Mainsail from the very first moment. It also notes, for each `EXCLUDE_OBJECT_START … EXCLUDE_OBJECT_END` block, **where that block begins and ends in the file**.
2. **While printing**, the file is read line by line. When a block belonging to an **excluded** object is reached, the reader **jumps directly to the end of that block** — nothing of it is executed, so no extrusion, travel or temperature change inside it happens.
3. The list of excluded objects is **written to a file**, so that after a **power loss** the cancellation is remembered when the print is resumed (chapter 21).
4. The list is cleared when the print file is reset (new print / end of print).

Consequences:
- Cancelling an object that is **currently being printed** takes effect at the **next START/END boundary**, not instantly **(to verify)**.
- Anything **between** the START and END lines is skipped, **including** non-extruding commands the slicer put there (e.g. a tool change). Normal slicer output keeps them outside the blocks.
- The firmware does not compensate the extruder position for skipped blocks beyond what the slicer's absolute/relative extrusion gives. **Relative extrusion (M83) is the safe case**; with absolute extrusion the slicer's `G92 E0` resets normally make it work too **(to verify)**.
- The commands `EXCLUDE_OBJECT_START/END` only matter when the line begins exactly with that word at the start of the line (the file reader looks at the line start).

---

## `EXCLUDE_OBJECT_DEFINE` — Declare an object

### What it does

1. With `NAME` (and optional `CENTER`, `POLYGON`): adds an object to the list (kept sorted by name). The centre and outline are only used for display (Mainsail's / the screen's object map).
2. With `RESET=1`: **forgets all objects and exclusions** and deletes the saved power-loss copy of the excluded list.
3. With nothing: lists the known objects in the console (`JSON=1` → full details as JSON).

### Syntax

```
EXCLUDE_OBJECT_DEFINE NAME=<name> [CENTER=<x>,<y>] [POLYGON=[[x,y],[x,y],…]]
EXCLUDE_OBJECT_DEFINE RESET=1
EXCLUDE_OBJECT_DEFINE [JSON=1]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `NAME` | Object name (the identifier used by all other commands). Case and non-ASCII letters are kept as written (the stock slicer produces names such as `3DBenchy_id_0_copy_0`). | Text without spaces | — |
| `CENTER` | Centre of the object on the plate. | `x,y` in mm | none |
| `POLYGON` | Outline of the object. | JSON list of `[x,y]` points, in mm | none |
| `RESET` | Clear everything. | any value | — |
| `JSON` | With no `NAME`: print the object list in JSON. | any value | plain names |

### Examples

```
EXCLUDE_OBJECT_DEFINE NAME=Cube_1 CENTER=100,100 POLYGON=[[90,90],[110,90],[110,110],[90,110]]
```
Declares a 20 mm square object.

```
EXCLUDE_OBJECT_DEFINE
```
Console: `Known objects: [...]`.

### Notes and cautions

- A badly formed `CENTER` or `POLYGON` makes the command **fail with an error** (the print stops with an error if it comes from the file).
- Declaring the same name twice creates **two entries** with that name.

---

## `EXCLUDE_OBJECT_START` / `EXCLUDE_OBJECT_END` — Mark an object's G-code

### What they do

- `EXCLUDE_OBJECT_START NAME=<n>`: makes `<n>` the **current object**. If `<n>` was not declared with `EXCLUDE_OBJECT_DEFINE`, it is added to the list (name only, no outline).
- `EXCLUDE_OBJECT_END NAME=<n>`: no more current object. If `<n>` differs from the current object's name, the mismatch is only written to the log; the current object is cleared anyway.

Both are normally written by the slicer around each object. You normally never type them.

### Syntax

```
EXCLUDE_OBJECT_START NAME=<name>
EXCLUDE_OBJECT_END NAME=<name>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `NAME` | Object name. **Required.** | Text | — |

### Notes and cautions

- The actual skipping is performed by the print-file reader (see "How it works"), which recognises the START/END **lines of the file**. Sending `EXCLUDE_OBJECT_START` by hand from the console or from a macro sets the current object but does not make the printer skip anything.

---

## `EXCLUDE_OBJECT_END_NO_OBJ` — End the current object without a name

If an object is current, behaves exactly as `EXCLUDE_OBJECT_END NAME=<current>`. Otherwise does nothing. Used for purge towers, prime lines and other global features that follow an object. No parameters.

```
EXCLUDE_OBJECT_END_NO_OBJ
```

---

## `EXCLUDE_OBJECT` — Cancel or restore an object

### What it does

| Call | Effect |
|---|---|
| `EXCLUDE_OBJECT NAME=<n>` | **Cancels** object `<n>`: its remaining G-code will be skipped. Already excluded → nothing more. |
| `EXCLUDE_OBJECT CURRENT=1` | Cancels the **current object** (the one between the last START and END). If there is none, a warning goes to the log. |
| `EXCLUDE_OBJECT RESET=1 NAME=<n>` | **Restores** `<n>` (removes it from the excluded list). |
| `EXCLUDE_OBJECT RESET=1` | Restores **all** objects. |
| `EXCLUDE_OBJECT` | Lists the excluded objects in the console (`Excluded objects: …`). |

The list of excluded objects is saved so it survives a power loss (see above).

### Syntax

```
EXCLUDE_OBJECT [NAME=<name>] [CURRENT=1] [RESET=1]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `NAME` | Object to cancel (or restore, with `RESET`). | An object name | — |
| `CURRENT` | Cancel whatever is currently printing. | any value | — |
| `RESET` | Restore instead of cancel (`NAME` = one object, otherwise all). | any value | — |

### Examples

```
EXCLUDE_OBJECT NAME=3DBenchy_id_0_copy_0
```
Cancels that part; the printer keeps printing the others.

```
EXCLUDE_OBJECT RESET=1 NAME=3DBenchy_id_0_copy_0
```
Takes the cancellation back (only useful **before** the printer has passed that part).

### Notes and cautions

- **Cancelling is not reversible for parts already skipped**: restoring afterwards does not print what was skipped.
- The name must match an object exactly. An unknown name is accepted without an error but has no effect **(to verify)**.
- The messages `Excluding object …` / `Unexcluding object …` are written to the **log only**, not to the console (unlike standard Klipper). Use `EXCLUDE_OBJECT` without parameters to check.

---

## `EXCLUDE_OBJECT_SET_OBJ` — Declare N anonymous objects (Anycubic)

### What it does

Runs `EXCLUDE_OBJECT_DEFINE NAME=0`, `NAME=1`, … `NAME=<T-1>`: *T* objects named by number, without outline. Meant for slicers that number objects (Marlin's `M486`).

### Syntax

```
EXCLUDE_OBJECT_SET_OBJ T=<count>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `T` | Number of objects. | Integer | — |

---

## `M486` — Marlin-style cancel-object (macro of `printer.cfg`)

`M486` is not built in: it is a **macro** in the stock `printer.cfg` that translates the Marlin/OctoPrint syntax into the commands above. Several options can be combined in one call; they run in this order:

| Option | What it does | Translates to |
|---|---|---|
| `T<n>` | Resets everything and declares *n* objects (`0` … `n-1`). | `EXCLUDE_OBJECT RESET=1` + `EXCLUDE_OBJECT_SET_OBJ T=<n>` |
| `C` | Cancels the current object. | `EXCLUDE_OBJECT CURRENT=1` |
| `P<i>` | Cancels object number *i*. | `EXCLUDE_OBJECT NAME=<i>` |
| `S<i>` | Starts object *i*; `S-1` means "no object". | `EXCLUDE_OBJECT_START NAME=<i>` / `EXCLUDE_OBJECT_END_NO_OBJ` |
| `U<i>` | Un-cancels object *i*. | `EXCLUDE_OBJECT RESET=1 NAME=<i>` |

`A` (object label) is ignored. For `S`, the file reader's block-skipping works only for lines that begin with `EXCLUDE_OBJECT_START` — **an `M486 S…` line in the file marks the current object but does not give the skip-ahead behaviour** **(to verify)**; use the `EXCLUDE_OBJECT_*` form in slicers.

```
M486 T3
M486 S0
; … object 0 …
M486 S-1
M486 P1
```
Declares three objects, prints object 0, then cancels object 1.

---

## Notes common to the chapter

- Anycubic Slicer Next writes `EXCLUDE_OBJECT_DEFINE … CENTER= POLYGON=` and START/END lines (confirmed in the sample files). `3DBenchy_18m.gcode` has none, so object cancelling is unavailable for it.
- Object names with spaces would break the parameter parsing; slicers use `_` instead.

---

*End of chapter 6.*

---

# Chapter 7 — Extruder

Nozzle temperature, pressure advance, and the low-level settings of the extruder motor. The Kobra 3 Max has **one** extruder (`extruder`), so the "which extruder" parameters of standard Klipper are of little use here.

| Command | In one line |
|---|---|
| `M104` | Sets the nozzle temperature and continues immediately. |
| `M109` | Sets the nozzle temperature and **waits** until it is reached. |
| `M900` | Sets pressure advance (`K`) and its smoothing time (`S`). |
| `SET_PRESSURE_ADVANCE` | Same as `M900` with Klipper-style parameter names. |
| `ACTIVATE_EXTRUDER` | Selects the active extruder (nothing to switch on this printer). |
| `SET_EXTRUDER_DIR` | Inverts the direction of the extruder motor. |
| `SET_EXTRUDER_ROTATION_DISTANCE` | Changes the extruder calibration (mm of filament per motor turn). |
| `SET_EXTRUDER_STEP_DISTANCE` | Same, expressed per micro-step. |
| `SYNC_STEPPER_TO_EXTRUDER` / `SYNC_EXTRUDER_MOTION` | Make an extra extruder motor follow the extruder (not used here). |

---

## `M104` / `M109` — Set the nozzle temperature

### What they do

1. Read the target `S` (°C).
2. **Cap it at the heater's `max_safe_temp`** (stock: **300 °C**, although the heater's absolute limit `max_temp` is 320). A higher request is **silently lowered to 300**, no error.
3. Apply the optional `I` floor (below).
4. Set the new target. If the result is outside the heater's allowed range (`min_temp` … `max_temp`, stock −200 … 320) the command fails with *"Requested temperature (…) out of range (…)"* — with the 300 °C cap this can in practice only happen for very low values.
5. **`M104`** returns at once, the nozzle heats in the background. **`M109`** then **waits until the nozzle has reached the target** (the printer keeps handling other things, the console may show temperature reports). `M109` with `S0` does not wait (it is then just "heater off").

### Syntax

```
M104 [S<°C>] [I<0|1>]
M109 [S<°C>] [I<0|1>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `S` | Target temperature. `0` switches the heater off. | °C, capped at 300 (stock) | 0 |
| `I` | **Anycubic option: "raise to the minimum safe temperature".** See below. | `0` = off, `1` (or any positive integer) = on | 0 |

There is **no `T` parameter** (tool number): the command always acts on the single extruder, `T1` etc. are ignored.

### The `I` parameter (answer to open question 1)

The option exists only on `M104` / `M109` (the bed commands `M140` / `M190` never pass it, see chapter 11). In the code, when `I` is **greater than 0** and `S` is **below a floor value**, `S` is **raised to that floor**:

- if the print-statistics state is exactly **`"heating"`**: the floor is the heater's **`min_safe_temp`** (170 °C nozzle, 35 °C bed on the stock configuration);
- in any other state: the floor is the heater's `min_temp` (−200 °C on the stock configuration).

**But the state `"heating"` is never set anywhere in the firmware.** The print-statistics state is only ever set to `printing`, `paused`, `complete`, `error`, `cancelled` and `standby` (checked by searching the whole program for any code that writes that word; the live printer reports `printing` during preheating). The only place where `"heating"` appears is this comparison. So **in practice `I1` changes nothing**: the floor is always `min_temp`, far below any real target. `G9111`, `G9112` and `PA_AUTO_CALIBRATE` pass `I1`, but their temperatures are unaffected. (It may be a leftover of a feature that another firmware version or a different Anycubic component set.)

### Examples

```
M104 S200
```
Start heating the nozzle to 200 °C and carry on.

```
M109 S215
```
Heat to 215 °C and wait for it.

```
M104 S350
```
Heats to **300 °C** (silent cap).

### Notes and cautions

- The heater's `min_extrude_temp` (stock 170 °C) is the temperature below which extrusion is normally refused (standard Klipper behaviour; not traced here).
- A shutdown (`M112`/`M8844`, heater fault) ends the wait.
- Raising `max_safe_temp` requires editing `printer.cfg`; do it only if your hotend/PTFE allow it.

---

## `M900` / `SET_PRESSURE_ADVANCE` — Pressure advance

Pressure advance compensates the elasticity of the filament (see chapter 4). Both commands set the same two values; `M900` is the Marlin-style spelling (`K` = advance, `S` = smoothing time), `SET_PRESSURE_ADVANCE` the Klipper spelling.

### What they do

1. Read the **smooth time** (`S` / `SMOOTH_TIME`): limits 0 – 0.2 s; if absent the **current value is kept**.
2. Read the **advance** (`K` / `ADVANCE`): limits 0 – 0.2 s; if absent the **current value is kept**. Out-of-range values give a "must have minimum/maximum" error.
3. **Exception — a successful calibration overrides `K`:** if `PA_AUTO_CALIBRATE` has succeeded for the current filament (status `1`) and `enable_dyna_PA` is off (the stock setting), `K` / `ADVANCE` is **not read at all**: the stored calibrated value is used instead (limited to 0 – 0.2). The smooth time is still read. The condition, how long it lasts (until the end or cancel of the next print, a firmware restart, or a failed re-calibration) and the dynamic mode are described in chapter 4, "Static and dynamic calibration in detail". In practice: **after a successful `PA_AUTO_CALIBRATE`, your own `M900 K…` (or one in the slicer's G-code) has no effect until the calibration is cleared.**
4. Applies the values at once to the extruder; they affect the moves queued after the command.
5. **Nothing is printed** to the console (contrary to standard Klipper). The current values appear in Mainsail's status or in the object `extruder` (`pressure_advance`, `smooth_time`; live printer: 0.02 and 0.04).

### Syntax

```
M900 [K<advance>] [S<smooth time>]
SET_PRESSURE_ADVANCE [ADVANCE=<advance>] [SMOOTH_TIME=<seconds>] [EXTRUDER=extruder]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `K` / `ADVANCE` | Pressure advance, in seconds of extruder motion per mm/s of speed. `0` switches it off. | 0 – 0.2 | current value |
| `S` / `SMOOTH_TIME` | Time window over which the advance is smoothed. | 0 – 0.2 s | current value |
| `EXTRUDER` | (`SET_PRESSURE_ADVANCE` only) extruder name. | `extruder` | the active extruder |

Stock configuration: `pressure_advance: 0.02`; smooth time 0.04 s (default of the firmware).

### Examples

```
M900 K0.04
```
Sets `K` to 0.04, keeps the smooth time.

```
SET_PRESSURE_ADVANCE ADVANCE=0.03 SMOOTH_TIME=0.04
```
Same thing in Klipper syntax.

### Notes and cautions

- The setting lasts until changed or the printer restarts; at start the value of `pressure_advance` in `printer.cfg` is loaded again.
- `PA_AUTO_CALIBRATE` (chapter 4) sends `M900` for you.
- A slicer's start G-code may override your value; check what yours sends.

---

## `ACTIVATE_EXTRUDER` — Select the active extruder

If `EXTRUDER=<name>` is already the active extruder the console says *"Extruder <name> already active"*. Otherwise the printer finishes the queued moves, announces *"Activating extruder <name>"* and switches. With a single extruder the only valid call is:

```
ACTIVATE_EXTRUDER EXTRUDER=extruder
```
→ `Extruder extruder already active`. Nothing else happens.

---

## `SET_EXTRUDER_DIR` — Invert the extruder motor direction

### What it does

Waits for queued moves, then sets the direction of the extruder motor: normal or inverted compared with the configuration. **If the motor now runs backwards, extrusion becomes retraction and vice versa.**

### Syntax

```
SET_EXTRUDER_DIR [INVERT=<0|1>] [EXTRUDER=extruder]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `INVERT` | `1` = reverse the direction, `0` = as configured. | 0 or 1 | 0 |

### Notes and cautions

- Anycubic addition (not in standard Klipper). Only for testing; use `INVERT=0` to restore. A restart also restores the configuration.
- With `INVERT=1` the extruder pushes filament **out backwards**: do not print like this.

---

## `SET_EXTRUDER_ROTATION_DISTANCE` — Change extruder calibration

### What it does

1. Waits until the queued moves are finished.
2. Changes the **rotation distance**: millimetres of filament moved per full turn of the extruder motor (stock: **4.604715**).
3. A **negative** value uses the absolute value and **inverts the direction**.
4. Prints *"Extruder 'extruder' rotation distance set to <value>"*.

### Syntax

```
SET_EXTRUDER_ROTATION_DISTANCE DISTANCE=<mm> [EXTRUDER=extruder]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `DISTANCE` | New rotation distance. **Required.** | mm, positive; negative also inverts direction; **0 → error "Rotation distance can not be zero"** | — |

### Examples

```
SET_EXTRUDER_ROTATION_DISTANCE DISTANCE=4.5
```
A smaller number extrudes **more** filament per requested E mm. To correct: new = old × (length actually extruded ÷ length requested).

### Notes and cautions

- Temporary: lost at restart. Write the final value in `printer.cfg` (`rotation_distance`).
- Unlike standard Klipper, `0` does **not** restore the configured value; it is an error.

---

## `SET_EXTRUDER_STEP_DISTANCE` — Change extruder calibration (per step)

Same as the previous command, but the value is the distance of **one micro-step**: `rotation distance = DISTANCE × steps per turn` (stock 200 × 16 = 3200 steps/turn). Prints *"Extruder 'extruder' step distance set to <value>"*.

```
SET_EXTRUDER_STEP_DISTANCE DISTANCE=<mm>
```

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `DISTANCE` | Distance per micro-step. **Required.** | mm, **greater than 0** | — |

---

## `SYNC_STEPPER_TO_EXTRUDER` and `SYNC_EXTRUDER_MOTION` — Make a second extruder motor follow

### What they do

For printers with a **separate** extruder-stepper object (`[extruder_stepper …]`): make that motor follow the move queue of the named extruder (or detach it when the name is empty). Waits for queued moves first. An unknown name gives *"'<name>' is not a valid extruder"*; on success *"Extruder stepper now syncing with '<name>'"*.

```
SYNC_STEPPER_TO_EXTRUDER [STEPPER=<stepper name>] EXTRUDER=<extruder name>
SYNC_EXTRUDER_MOTION [EXTRUDER=<stepper name>] MOTION_QUEUE=<extruder name>
```

### Notes and cautions

- The stock `printer.cfg` of the Kobra 3 Max has **no** extra extruder stepper, so these commands have nothing to act on. They exist because the module is the standard Klipper one **(not exercised)**.

---

*End of chapter 7.*


---

# Chapter 8 — Factory / engineering (`CS1237_DUMP`, `G9121`, `G9122`, `G9123`, `PCBA_READ`, `PCBA_WRITE`)

> **Warning — factory commands.** These commands are meant for production-line testing and repair. None of them is needed for printing, calibration or normal maintenance. Two of them (`G9123`, `PCBA_WRITE`) **write data into the toolhead board**. Do not experiment with them on a printer you care about. Everything below comes from reading the firmware; nothing was run on a printer, and the factory test procedure itself is unknown, so the meaning of some values is **(to verify)**.

## Background: the CS1237 sensor

The nozzle (toolhead) board carries a **CS1237**, a small 24-bit analog-to-digital converter that reads a **strain gauge** (load cell). It is what makes the Kobra 3 Max's contact probing work: when the nozzle touches the bed (or something presses on the nozzle) the reading shifts, and the board reports it. The same sensor is also used to detect, while printing, that the nozzle has been **blocked / hit something**, that **filament is scratching** at the extruder, or that the **filament is blocked**.

In `printer.cfg` the `[cs1237]` section sets four trigger thresholds (negative numbers; the more negative, the less sensitive):

| Config key | Stock value | Used for |
|---|---|---|
| `sensitivity` | −2000 | normal probing |
| `head_block_sensitivity` | −300000 | "toolhead blocked" notice |
| `scratch_sensitivity` | −100000 | "scratch" notice |
| `self_check_sensitivity` | −400 | self-test |
| `block_filament_sensitivity` | −3000 | "filament blocked" notice |

The commands below start or stop the sensor's **continuous reporting** with one of these thresholds, or write its self-test flag. (The `sensitivity` value −2000 is sent to the board when it is configured at start-up; it is not selectable by these commands.)

| Command | In one line |
|---|---|
| `CS1237_DUMP` | Starts/stops sensor reporting with a threshold chosen by `S`. |
| `G9121` | Starts/stops sensor reporting with the **"scratch"** threshold. |
| `G9122` | Powers the sensor and starts/stops reporting with the **self-test** threshold. |
| `G9123` | Writes the sensor self-test flag in the toolhead board. |
| `PCBA_READ` | Reads the board's factory flag and serial number. |
| `PCBA_WRITE` | Writes the board's factory flag and serial number. |

---

## `CS1237_DUMP` — Start or stop sensor reporting

### What it does

1. Picks a threshold from `S` (table below).
2. If `E` is greater than 0: tells the toolhead board to **start reporting** sensor readings every `T` seconds, using that threshold and mode number.
3. If `E` is 0 (or negative): tells the board to **stop reporting** (the same message with reporting switched off).

The command itself prints nothing; the readings and the "notice" events (`head_block_notice`, `scratch_notice`, `block_filament_notice`) are handled inside the firmware. Help text: *"debug CS1237 DUMP"*.

### Syntax

```
CS1237_DUMP [E=<0|1>] [T=<seconds>] [S=<1|2|4|8>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `E` | Enable. `>0` = start reporting, otherwise stop. | integer | 0 (stop) |
| `T` | Reporting interval. | seconds (decimals allowed) | 0.1 |
| `S` | Mode / threshold selector. | `1` = self-test threshold, `2` = scratch threshold, `4` = filament-blocked threshold, `8` = toolhead-blocked threshold; any other value → threshold 0 | 1 |

### Examples

```
CS1237_DUMP E=1 S=2 T=0.05
```
Start reporting every 50 ms with the "scratch" threshold.

```
CS1237_DUMP
```
Stop reporting (`E` defaults to 0).

### Notes and cautions

- Mode `S` also tells the board which kind of event to watch for, so `S` is not just a threshold lookup **(to verify the exact board-side behaviour; it is in the MCU firmware, not in `gklib`)**.
- Starting reporting while a print or a probing run is using the sensor may interfere with it **(untested)**. Send `CS1237_DUMP` (stop) afterwards.

---

## `G9121` — Sensor reporting, "scratch" mode

Same as `CS1237_DUMP S=2`, with one extra option.

### What it does

1. If `E` > 0: start reporting every `T` seconds, mode 2, **scratch threshold**.
2. Otherwise: stop reporting for mode 2.
3. `R` > 0 additionally sets an internal "report" flag in the firmware (`R` ≤ 0 clears it). What that flag changes downstream is **(to verify)**.

### Syntax

```
G9121 [E=<0|1>] [R=<0|1>] [T=<seconds>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `E` | Enable (`>0` start, otherwise stop). | integer | 0 |
| `R` | Internal report flag. | `0` / positive | 0 |
| `T` | Reporting interval. | seconds | 0.1 |

Example: `G9121 E=1 T=0.2` starts, `G9121` alone stops.

---

## `G9122` — Power the sensor and report, self-test mode

### What it does

1. If `E` > 0:
   1. **Powers up the sensor** (sends the enable message to the board, then waits 0.5 s with `G4 P500`). The power-up is **reference-counted**, so several users can enable it and it only turns off when the last one releases it.
   2. Starts reporting every `T` seconds, mode 1, **self-test threshold** (−400 on the stock config).
2. Otherwise: stops reporting for mode 1, then **releases the sensor power** (one count).
3. `R` is read the same way as in `G9121` (kept for symmetry; effect **(to verify)**).

### Syntax

```
G9122 [E=<0|1>] [R=<0|1>] [T=<seconds>]
```

### Parameters

As for `G9121`.

### Cautions

- Every `G9122 E=1` must be paired with one `G9122` (stop). Calling start twice and stop once leaves the sensor powered.
- The 0.5 s wait uses a G-code `G4`, so the command only returns after it.

---

## `G9123` — Write the sensor self-test flag

### What it does

1. Reads `W` and the target board `MCU`.
2. Sends the board a message to **set the self-test flag to `W`** and waits for the board's answer (with a timeout).
3. Prints the answer to the console as `mcu_name:<name> test_flag:<n>`.
4. If the named MCU does not exist: error *"MCU [<name>] is not exist, supported mcu=<list>"*.

### Syntax

```
G9123 [W=<0|1>] [MCU=<name>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `W` | Flag value to write. Meaning of values is a factory convention **(to verify)**; `1` is presumably "self-test done/passed". | integer | 0 |
| `MCU` | Which microcontroller to talk to. | `NOZZLEMCU` (toolhead board); other names from `printer.cfg`, e.g. `mcu` | `NOZZLEMCU` |

### Cautions

- This **changes a value stored in the toolhead board**. Don't use it casually: the flag may be checked by the factory-test or the self-check routines.

---

## `PCBA_READ` — Read the board's factory flag and serial number

### What it does

1. Asks the board named by `MCU` for its stored PCBA information.
2. Waits for the answer (with a timeout).
3. Prints it in the console as a JSON line with three fields: `mcu_name`, `pcba_flag`, `pcba_sn`.
4. Unknown MCU → *"MCU <name> is not exist"*.

This is **read-only** and harmless.

### Syntax

```
PCBA_READ [MCU=<name>]
```

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `MCU` | Board to read. | `NOZZLEMCU`, … | `NOZZLEMCU` |

Example: `PCBA_READ` → something like `{"mcu_name":"nozzle_mcu","pcba_flag":…,"pcba_sn":"…"}` (exact wording **to verify**).

---

## `PCBA_WRITE` — Write the board's factory flag and serial number

### What it does

1. Reads the current content of the board (like `PCBA_READ`). If that read fails: error *"MCU read pcba info failed"* and nothing is written.
2. Combines the current content with your `FLAG` and `SN` (a parameter you leave out keeps its current value **(to verify; the default `-1` / `None` look like "unchanged" sentinels, but a `SN` left at `None` may also make the command refuse to write)**).
3. Sends the new flag and serial-number data to the board to be stored permanently.

### Syntax

```
PCBA_WRITE [FLAG=<n>] [SN=<serial>] [MCU=<name>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `FLAG` | Factory status flag. | integer | −1 (don't change **(to verify)**) |
| `SN` | Board serial number string. | text, no spaces | `None` |
| `MCU` | Board to write. | `NOZZLEMCU`, … | `NOZZLEMCU` |

### Cautions

- **Overwrites the board's identification.** A wrong serial or flag can be confusing for Anycubic service/after-sales tools, and the firmware offers no "undo" except writing the old values back. **Run `PCBA_READ` first and keep a copy of the output.**
- The write is not reversible by a reboot.

---

## Summary for the guide reader

| If you want to… | Use |
|---|---|
| …just print | none of these |
| …check what serial/flag the toolhead board carries | `PCBA_READ` (safe) |
| …watch the strain-gauge sensor trigger levels | `CS1237_DUMP` / `G9121` / `G9122` (start, then stop) |
| …change factory data | `PCBA_WRITE`, `G9123` — only if you know why |

*End of chapter 8.*


---

# Chapter 9 — Fans (`M106`, `M107`)

The Kobra 3 Max has three fans, but **only one is under G-code control**: the **part-cooling fan** (config section `[fan]`, on the toolhead board). The other two are automatic: the **extruder (hotend) fan** (`[heater_fan extruder_fan]`) runs by itself whenever the nozzle is warm, and the **controller fan** (`[controller_fan]`) runs at full speed while the motors/electronics are active and for 30 s after. Neither can be set with `M106`/`M107`.

| Command | In one line |
|---|---|
| `M106` | Sets the part-cooling fan speed. |
| `M107` | Switches the part-cooling fan off. |

---

## `M106` — Set the part-cooling fan speed

### What it does

1. Reads `S` (0–255; **255 if you omit it**) and converts it to a fraction: `S / 255`.
2. Does **not** change the fan immediately: the request is queued **in step with the movement queue**, so the fan changes at the moment the toolhead reaches that point of the G-code, not when the line is read. (Same behaviour as in a normal print: `M106` in the file takes effect with the moves around it.)
3. At that moment the fan setting is applied:
   - the fraction is multiplied by the fan's `max_power` (stock: not set → 1.0, i.e. 100 %) and limited to the range 0…`max_power`; so `S` above 255 simply gives full power, a negative `S` gives off;
   - if the result is below the fan's `off_below` (stock: not set → 0) it is treated as 0;
   - the fan is driven with a PWM signal of cycle time 2 ms (stock `cycle_time: 0.002`, i.e. 500 Hz);
   - **kick start:** when the fan goes from stopped to running (or jumps up by more than 50 % of full power) it is first driven at **full power for 0.5 s** (stock `kick_start_time: 0.5`), then drops to the requested speed. This helps it start at low speeds.
4. No message is printed.

### Syntax

```
M106 [S<0-255>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `S` | Fan speed. | 0 = off, 255 = full; values in between are proportional (e.g. 128 ≈ 50 %). Above 255 is capped at full, below 0 means off. | **255** |

There is **no `P` parameter** (fan number): any `P` is ignored, there is only one controllable fan.

### Examples

```
M106 S255
```
Part-cooling fan to full speed (this is what the `WIPE_NOZZLE` macro does before wiping).

```
M106 S128
```
About half speed (it will first burst at full power for 0.5 s if it was stopped).

```
M106
```
Same as `M106 S255` — **no `S` means full speed**, not "keep the current speed".

### Notes and cautions

- Because the change is queued with the moves, typing `M106 S0` in the console while the printer is idle takes effect at once; during a print it takes effect when the toolhead reaches that point of the queue (a few moves after, depending on look-ahead).
- **Pause / resume (`PAUSE`, `M600`, `M601`):** the firmware handles the fan itself. On pause it **remembers the current fan speed (0–255) and switches the fan off**. On resume, after the filament purge, it **sets the fan back to the remembered speed** (and sets it once more at the very end of the resume sequence). So the speed set by your slicer's `M106` is restored automatically; you do not need to add an `M106` to the resume G-code. Between the purge and the end of the resume sequence the fan may briefly run at a separate "sweep" speed (config key `sweep_fan_speed`, not present in the stock `printer.cfg`, so default **(to verify)**) while the nozzle is wiped/cooled for `delay_before_throw` seconds (stock 3). Caveats **(to verify)**: only the speed at the moment of the pause is remembered — an `M106` that is still in the look-ahead queue when the pause happens may be lost; and I did not trace what happens if you change the fan manually while paused (the remembered value overwrites it on resume).

---

## `M107` — Fan off

### What it does

Queues, the same way as `M106`, a request to set the part-cooling fan to **0**. Equivalent to `M106 S0`. No parameters.

### Syntax

```
M107
```

### Example

```
M107
```
Stop the part-cooling fan (typically in the end G-code).

---

*End of chapter 9.*


---

# Chapter 10 — Filament sensor (`QUERY_FILAMENT_SENSOR`, `SET_FILAMENT_SENSOR`)

These two commands belong to the generic **"filament switch sensor"** module (the Klipper `[filament_switch_sensor <name>]` section): a switch on a pin that tells whether filament is present, with optional "runout" and "insert" actions.

> **Important — probably nothing to act on in the stock configuration.** The commands only work on a sensor that was **declared by name** in the configuration (`[filament_switch_sensor my_sensor]`). The stock `printer.cfg` of the Kobra 3 Max declares **none**, and the live printer lists no such object. The filament detection of this printer is done by other modules (`[filament_hub]`, `[filament_tracker]` with its break/block pins, and the `RUNOUT_PAUSE` command of chapter 1), which do **not** use these two commands. So on a stock printer these commands will most likely just report an error. They become useful only if you add a `[filament_switch_sensor …]` section yourself (e.g. through the Rinkhals configuration) **(not tested)**.

| Command | In one line |
|---|---|
| `QUERY_FILAMENT_SENSOR` | Prints whether a given sensor currently sees filament. |
| `SET_FILAMENT_SENSOR` | Switches a given sensor's runout/insert handling on or off. |

---

## `QUERY_FILAMENT_SENSOR` — Is there filament?

### What it does

Looks up the sensor named by `SENSOR` and prints one line in the console:

- `Filament Sensor <name>: filament detected`
- `Filament Sensor <name>: filament not detected`

It only reports the state; it changes nothing. The same information is available as the `filament_detected` field of the sensor's status (and through the web-API endpoint `filament/query`).

### Syntax

```
QUERY_FILAMENT_SENSOR SENSOR=<name>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `SENSOR` | Name of the sensor, i.e. the word after `filament_switch_sensor` in the config. **Required.** | text | — |

### Example

```
QUERY_FILAMENT_SENSOR SENSOR=my_sensor
```
→ `Filament Sensor my_sensor: filament detected`

With an unknown or missing `SENSOR` the console shows a standard G-code error (exact wording **to verify**).

---

## `SET_FILAMENT_SENSOR` — Enable or disable a sensor

### What it does

1. Looks up the sensor named by `SENSOR`.
2. Sets its **enabled** flag to `ENABLE` (non-zero = on, 0 = off). No message is printed.
3. While a sensor is **disabled**, its runout and insert events are **ignored** (no pause, no custom G-code). The sensor still reads the pin, so `QUERY_FILAMENT_SENSOR` keeps working.

The setting is **not saved**: after a restart the sensor is enabled again (the config has no `enable` key for it, and the firmware never writes this setting to `printer.cfg`) **(to verify)**.

### Syntax

```
SET_FILAMENT_SENSOR SENSOR=<name> [ENABLE=<0|1>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `SENSOR` | Sensor name. **Required.** | text | — |
| `ENABLE` | `1` = react to runout/insert, `0` = ignore them. | integer | **1** |

Note the default: `SET_FILAMENT_SENSOR SENSOR=my_sensor` **without** `ENABLE` **enables** the sensor.

### Examples

```
SET_FILAMENT_SENSOR SENSOR=my_sensor ENABLE=0
```
Temporarily ignore runout events (for example while changing filament by hand).

```
SET_FILAMENT_SENSOR SENSOR=my_sensor ENABLE=1
```
React again.

---

## What a sensor does when it is declared (for reference)

Config keys read for each `[filament_switch_sensor <name>]` (besides `switch_pin`):

| Key | Meaning | Default |
|---|---|---|
| `pause_on_runout` | When filament disappears **during a print**, run `PAUSE` (chapter 20) first. | true |
| `runout_gcode` | G-code to run after a runout (after the pause, if any). | none |
| `insert_gcode` | G-code to run when filament is inserted (when **not** printing). | none |
| `pause_delay` | Seconds between the `PAUSE` and `runout_gcode`. | 0.5 |
| `event_delay` | Minimum seconds between two events (debounce). | 3.0 |

Behaviour: a change of the switch is noted; events closer than `event_delay` to the previous one, or while the sensor is disabled, are dropped; a **runout while printing** triggers the pause and `runout_gcode`, an **insertion while not printing** triggers `insert_gcode`. The log line is *"Filament Sensor <name>: runout event detected, Time …"* / *"…insert event detected…"*. Anycubic's version also builds a small status record (state, printing time) at each event, whose use I could not determine **(to verify)**.

---

*End of chapter 10. Next chapter in the command table: **Heaters & temperature** (`M105 M140 M190 SET_HEATER_TEMPERATURE TEMPERATURE_WAIT TURN_OFF_HEATERS`).*


# Chapter 11 — Heaters & temperature (`M105`, `M140`, `M190`, `SET_HEATER_TEMPERATURE`, `TEMPERATURE_WAIT`, `TURN_OFF_HEATERS`)

Bed temperature, generic heater control, temperature reports and "wait for a temperature". The nozzle commands `M104` / `M109` are in chapter 7. The printer has two heaters, named **`extruder`** (nozzle) and **`heater_bed`** (bed), plus a few read-only temperature sensors (for example the controller board and the host CPU).

Stock limits (from `printer.cfg`):

| | Nozzle (`extruder`) | Bed (`heater_bed`) |
|---|---|---|
| Highest accepted target (`max_safe_temp`) | **300 °C** | **100 °C** |
| Absolute limit (`max_temp`) | 320 °C | 110 °C |
| Lowest allowed value (`min_temp`) | −200 °C | −200 °C |

| Command | In one line |
|---|---|
| `M105` | Reports the current and target temperatures. |
| `M140` | Sets the bed temperature and continues immediately. |
| `M190` | Sets the bed temperature and **waits** until it is reached. |
| `SET_HEATER_TEMPERATURE` | Sets the target of any heater by name (`extruder` or `heater_bed`). |
| `TEMPERATURE_WAIT` | Waits until any sensor is above and/or below a value. |
| `TURN_OFF_HEATERS` | Switches off nozzle and bed heaters. |

---

## `M105` — Report temperatures

### What it does

1. Reads the current temperature and the target temperature of every heater or sensor that has a G-code identifier.
2. Answers with one entry `<id>:<current> /<target>` per sensor (one decimal), separated by spaces, **sorted alphabetically by identifier**. The identifiers are **`B`** for the bed and **`T0`** for the nozzle (the firmware builds the nozzle's name as `T` + extruder number 0; the host-CPU and MCU sensors have no identifier in the stock config, so they are not listed). Expected reply on the stock printer:
   `B:23.9 /0.0 T0:24.8 /0.0` — **bed first, nozzle labelled `T0` (not `T` as in Marlin)**.
3. Special case: if the printer has not finished starting up, the answer is just `T:0`.
4. The line is sent back as the command's acknowledgement, so any host or console sees it.

Nothing is changed on the printer.

### Syntax

```
M105
```

### Parameters

None. Any parameter is ignored (standard `M105` has none either).

### Examples

```
M105
```
Ask for the temperatures. Mainsail does this by itself every second or so through its own channel; you only need `M105` in a terminal or a script.

### Notes and cautions

- Anycubic's firmware does not add an "ADC value" or power figure to the report.
- Verified in the program: the format string `%s:%.1f /%.1f`, the sorting, the `T0` / `B` identifiers (the strings `T%d` and `B` are passed when the nozzle and bed heaters are created) and the `T:0` fallback. Not seen on the real printer (whether the host prefixes `ok ` is up to the transport; **no live `M105` was sent**).
- If a heater has no usable sensor the firmware logs an error in its log (not in the console) and that heater is left out.

---

## `M140` / `M190` — Set the bed temperature

### What they do

1. Read the target `S` (°C). **Default 0**: `M140` or `M190` with no `S` **switches the bed off**.
2. Cap it at the bed's `max_safe_temp` (stock **100 °C**). A higher value is **silently lowered** to 100, no error.
3. Queue the change *behind the moves already planned*: the new target is applied when the printer has reached that point in the move queue, so it is correctly ordered with the surrounding G-code (standard Klipper behaviour for temperatures).
4. If the value is outside the heater's range (`min_temp` … `max_temp`, stock −200 … 110) the command fails with *"Requested temperature (…) out of range (…)"*. With the 100 °C cap this only happens for extremely low values.
5. **`M140`** returns at once; the bed heats in the background.
   **`M190`** then **waits until the bed is "close enough"**: the measured bed temperature must be **within ±2 °C of the target** (the wait ends as soon as it gets inside this window; it does not have to stay there for a while). While waiting, the printer re-checks about **once per second**. The wait also ends if the printer shuts down (heater fault, `M112`).
   `M190` with `S0` does not wait.

### Syntax

```
M140 [S<°C>]
M190 [S<°C>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `S` | Target bed temperature. `0` = heater off. | °C, capped at 100 (stock) | 0 |

There is **no `R` parameter** (Marlin's "wait while cooling down" form) and **no `I` parameter**: unlike `M104`/`M109` (chapter 7) the bed commands never pass an `I` flag to the heater code. *(Correction: an earlier sentence in chapter 7 said that `M190` has the same `I` option; it does not.)*

### Examples

```
M140 S60
```
Start heating the bed to 60 °C and carry on (typical: heat the bed while the nozzle is being wiped).

```
M190 S60
```
Heat to 60 °C and wait until the bed is between 58 and 62 °C.

```
M140 S0
```
Switch the bed heater off.

```
M190 S120
```
Heats to **100 °C** (silent cap).

### Notes and cautions

- Because the end of the wait is "±2 °C", `M190 S60` returns when the bed reaches about 58 °C if it comes from below, and the temperature then keeps rising to 60. For a really settled bed add a `G4 P60000` (one minute) afterwards. **(to verify on the printer: the ±2 °C window is read from the firmware's regulator code, not measured)**
- A **cooling** `M190` (for example from 100 °C to 60 °C) also waits only until the bed is within 2 °C of the new target; the heater stays off while it cools.
- `M190` has **no time limit**: if the bed never reaches the target (broken sensor, heater fault) the printer relies on its safety monitor (`verify_heater`, stock: error after the temperature fails to rise by 1 °C within 300 s, bed / 30 s, nozzle) to stop with an error.

---

## `SET_HEATER_TEMPERATURE` — Set any heater by name

### What it does

1. Looks up the heater named in `HEATER`.
2. Reads `TARGET` (°C, default 0) and applies it **exactly like `M140`/`M104`** (same cap at `max_safe_temp`, same range check, same queuing behind pending moves).
3. **Never waits** — it is the "start heating and carry on" form.

### Syntax

```
SET_HEATER_TEMPERATURE HEATER=<extruder|heater_bed> [TARGET=<°C>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `HEATER` | Which heater. **Required.** | `extruder`, `heater_bed` | — |
| `TARGET` | Target temperature. `0` switches the heater off. | °C, capped at the heater's `max_safe_temp` (nozzle 300, bed 100) | 0 |

### Examples

```
SET_HEATER_TEMPERATURE HEATER=extruder TARGET=200
```
Same as `M104 S200`.

```
SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET=0
```
Bed off.

### Notes and cautions

- `TARGET` omitted = **off**, not "keep the old value".
- A missing or unknown `HEATER` is rejected by the command dispatcher (standard Klipper wording about the missing/unknown parameter; **exact message to verify**).
- This command has no `I` option; `M104`/`M109` are the only commands that do (and it does nothing, see chapter 7).

---

## `TEMPERATURE_WAIT` — Wait until a sensor is in a range

### What it does

1. Looks up the sensor named in `SENSOR`. Any heater or temperature sensor known to the printer works (`extruder`, `heater_bed`, and the sensors configured with `[temperature_…]` sections such as the host or MCU sensors). If the name is unknown the command fails with *"Unknown sensor …"*.
2. Reads `MINIMUM` and `MAXIMUM`. **In this firmware the command is broken and never returns** (see the warning below): the value of `MAXIMUM` is limited to `MINIMUM` (the port of Klipper's "`MAXIMUM` must be above `MINIMUM`" rule was put in the wrong slot of `Get_float`, so it caps the value instead of checking it; the cap is applied silently: the error is only logged and the limit is used, ch. 24). The two defaults are both parsed from the text `-inf`. If neither is given it only writes *"Error on TEMPERATURE_WAIT: missing MINIMUM or MAXIMUM."* to the log and goes on to wait.
3. **Checks the temperature about once per second** until it is `≥ MINIMUM` **and** `≤ MAXIMUM`, then returns. It does not change any target and does not heat or cool anything by itself.
4. Ends early if the printer shuts down.

### Syntax

```
TEMPERATURE_WAIT SENSOR=<name> [MINIMUM=<°C>] [MAXIMUM=<°C>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `SENSOR` | The sensor or heater to watch. **Required.** | `extruder`, `heater_bed`, or a configured sensor name | — |
| `MINIMUM` | Wait until the temperature is **at least** this. | °C | −∞ |
| `MAXIMUM` | Wait until the temperature is **at most** this. | °C | −∞ (not +∞ as in Klipper) |

> **Warning — do not use `TEMPERATURE_WAIT` on this firmware.** The wait ends only when `MINIMUM ≤ temperature ≤ MAXIMUM`, but `MAXIMUM` is silently capped at `MINIMUM` (and at −∞ when `MINIMUM` is not given), so the window is empty (or a single exact value) and the command never returns. **Observed on a printer:** `TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM=45` and `… MINIMUM=45 MAXIMUM=150` in a print start code both left the printer stuck until it was power-cycled. The cap was read in the disassembly of `Cmd_TEMPERATURE_WAIT` and `Get_float` (the second `Get_float` call passes the `MINIMUM` variable in the *maximum* slot). To wait for a temperature use `M109` / `M190` (they wait until the temperature is within ±2 °C of the target, also when cooling). The examples below show the intended syntax and do not work as described.

### Examples

```
TEMPERATURE_WAIT SENSOR=extruder MINIMUM=190
```
Intended meaning: wait until the nozzle is at 190 °C or more (after e.g. `M104 S200`), without being tied to the heater's own target. **On this firmware it never returns** (see the warning above); use `M109 S200` instead.

```
TEMPERATURE_WAIT SENSOR=heater_bed MAXIMUM=40
```
Wait until the bed has cooled down to 40 °C or less (typical at the end of a print before removing the part).

```
TEMPERATURE_WAIT SENSOR=extruder MINIMUM=195 MAXIMUM=205
```
Wait until the nozzle is between 195 and 205 °C.

### Notes and cautions

- **No time limit.** If the temperature can never reach the range (for example `MAXIMUM=40` while the bed heater is on at 60 °C) the print waits forever. Use `M112`/the emergency stop or `CANCEL_PRINT` to get out.
- It is read **from the sensor directly**, not synchronised with the move queue like `M190`. Put an `M400` before it if you want it to start only after the moves that precede it have finished. **(to verify)**
- Whether a *missing* `SENSOR=` gives a "missing parameter" error or is treated as an unknown sensor was not traced.

---

## `TURN_OFF_HEATERS` — Switch off all heaters

### What it does

1. Sets the target of **every** heater (nozzle and bed) to 0 °C.
2. Nothing else: fans keep running (the extruder fan switches itself off a while after the nozzle has cooled, the part-cooling fan is not touched), the motors stay enabled, the print is not stopped.

### Syntax

```
TURN_OFF_HEATERS
```

### Parameters

None.

### Examples

```
TURN_OFF_HEATERS
```
Typical in a slicer's end G-code, or after cancelling a print.

### Notes and cautions

- It sets the targets to 0 directly; it does not wait and does not report anything.
- Use `M107` (chapter 9) to also stop the part-cooling fan.

---

*End of chapter 11. Next chapter in the command table: **Homing** (`G28 HOME_XY HOME_Y` — including the meaning of `G28 W`).*


# Chapter 12 — Homing (`G28`, `HOME_XY`, `HOME_Y`)

The printer finds its position by homing: X and Y by driving into the frame (the stepper drivers detect the stall — no switches), Z with the nozzle's pressure sensor (the "probe"). On this firmware `G28` is **wrapped by a "safe Z home" routine** (`[safe_z_home]` in `printer.cfg`) that lifts the nozzle first, homes X/Y, drives to a safe spot and only then homes Z.

Stock values used below (`printer.cfg`):

| Setting | Value | Meaning |
|---|---|---|
| `home_xy_position` | 210, 210 | Where Z is homed (bed centre) |
| `z_hop` / `z_hop_speed` | 15 mm / 8 mm/s | Lift before homing |
| `speed` | 200 mm/s | Speed of the travel to the Z-homing spot |
| `custom_home_xy_re_position` | 10, 10 | Step away from the corner after `HOME_XY` |
| `move_to_previous` | not set (= off) | Do not return to the old XY after Z homing |
| X homing | towards X = −28, 30 mm/s, retract 30 mm | stall detection |
| Y homing (two motors `y`, `y1`) | towards Y = −5.5, 25 mm/s, retract 30 mm | stall detection |
| Z homing | probe (nozzle sensor), 6 mm/s, then 3 mm/s, retract 4 mm | |

| Command | In one line |
|---|---|
| `G28` | Homes all axes, or only the ones named (`X`, `Y`, `Z`); `W`, `L`, `R` pick a different spot for Z homing. |
| `HOME_XY` | Homes X and Y, then steps 10 mm away from the corner. |
| `HOME_Y` | Homes Y, then steps 15 mm away from the rear/front stop. |

---

## `G28` — Home axes

### What it does

1. **Starts a fresh run** (cancel marker cleared; see notes).
2. **Z lift (only if `z_hop` is not 0; stock 15 mm):**
   - If Z is **not homed**: the firmware assumes Z = 0 and **lifts the nozzle by `z_hop` (15 mm) at 8 mm/s**, so the nozzle cannot drag on the bed while X/Y move. (The lift is skipped if the remembered Z is **above `position_max` − 20 mm** — stock 502 − 20 = 482 mm — because lifting further would hit the top; Z is still flagged "not homed". After a restart the remembered Z is 0, so in practice the lift happens.) Z is then flagged as "not homed".
   - If Z **is** homed and lower than `z_hop`: it is lifted up to `z_hop`.
3. **Decides which axes to home.** `X`, `Y`, `Z` select axes; if **none** of the three is given, all three are homed. The options `W`, `L`, `R` do **not** count as axes, so `G28 W` still homes everything.
4. **X and Y** (if wanted): homed together in one standard Klipper homing run (X first, then Y; each: fast approach, retract, slow second approach).
5. **Z** (if wanted):
   1. If X or Y are **not homed yet**, `HOME_XY` is run first (so the stepping-away move of `HOME_XY` is included).
   2. **Moves the nozzle to the Z-homing spot** at `speed` (200 mm/s) and waits for the move to end. The spot is the bed centre (210, 210) unless `W`, `L` or `R` is given (see below).
   3. **Homes Z** with the nozzle sensor (down until the sensor triggers, retract 4 mm, slow second approach).
   4. If `z_hop` is not 0: **lifts the nozzle to 15 mm** again.
   5. If `move_to_previous` is on (stock: off): returns to the X/Y position it had before. Otherwise the nozzle stays at the Z-homing spot.
6. If only X and/or Y were requested, step 5 is skipped completely — **Z is not touched** (but the lift of step 2 is done).
7. A *homing end* notification is sent to the rest of the firmware.

### Syntax

```
G28 [X] [Y] [Z] [W] [L] [R]
```

The axis letters are **flags without values** (`G28 X`, not `G28 X0`).

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `X` | Home X. | flag | — |
| `Y` | Home Y. | flag | — |
| `Z` | Home Z. | flag | — |
| *(none of X, Y, Z)* | Home all three. | — | all axes |
| `W` | **Z-homing spot = "platform" (nozzle-wiping pad) position of the LeviQ3 auto-level system, minus 10 mm in Y.** Stock `platform_offset: 30,414` → Z is homed at **X = 30, Y = 404**, i.e. at the rear-left wiper area instead of the bed centre. Used by `G9112` before its measurements. | flag | off |
| `L` | Z-homing spot **X = 15, Y = 125** (fixed numbers). Only used by a factory test (one of `G9100`–`G9104`). | flag | off |
| `R` | Z-homing spot **X = 235, Y = 125** (fixed numbers). Same factory test. | flag | off |

Details for `W`, `L`, `R`:
- They only change **where Z is homed**; they have an effect only if Z is homed in that call (always true for a bare `G28 W`).
- They need the LeviQ3 module to exist (it does on this printer); without it they are silently ignored and the bed centre is used.
- If several are given, **`R` wins over `L`, `L` over `W`**.
- `L` and `R` use the numbers of the smaller Kobra 3 (250 mm bed) — on the Max they are a point in the front-left/centre part of the bed, not the edges.

### Examples

```
G28
```
Home everything: lift 15 mm, X, Y, move to the bed centre, Z, lift 15 mm again. The nozzle ends at (210, 210), Z = 15.

```
G28 X Y
```
Home X and Y only. Z is not homed (but it is lifted by 15 mm first).

```
G28 Z
```
Home Z only. If X/Y are not homed yet, they are homed first (with `HOME_XY`).

```
G28 W
```
Home everything, with Z measured at the wiper-pad position (30, 404). The nozzle must be clean and **the area under it free** — Z homing presses the nozzle onto whatever is there.

### Notes and cautions

- **Z is homed by pressing the nozzle onto the surface.** Heat the nozzle only as needed and wipe it first (`G9111` does this before a print); a blob of plastic on the nozzle gives a wrong Z.
- **Home X/Y first when the toolhead may be anywhere**: `G28 Z` is allowed, but it will home X/Y first if they are not homed yet.
- **Cancel:** when you press **Cancel** in the web interface or on the touchscreen (the `pause_resume/cancel` request), the firmware first announces the cancel to all modules and then runs `TURN_OFF_HEATERS` and `CANCEL_PRINT`. `G28` listens for that announcement: it is checked after the Z lift, after the X/Y homing, after the travel to the Z-homing spot and after the Z homing. If a cancel arrived, the homing stops at the next check with the error *"Instruction cancellation"* (the move already running is not interrupted — it ends first). Typing `CANCEL_PRINT` yourself as a G-code does **not** send that announcement, so it does not stop a `G28` already running. A cancel that arrives while no `G28` runs has no effect on the next `G28` (the marker is cleared when `G28` starts).
- If a homing move fails (stall not found, etc.) the motors are switched off by the homing code and the error is reported; the printer then counts as "not homed". **(to verify: motor-off is in the homing error handler, seen in the call list)**
- `G28` does not change temperatures or the bed mesh, and does not run `BED_MESH_CALIBRATE`.
- Standard Klipper's `G28` has no `W`/`L`/`R`; they are Anycubic additions.

---

## `HOME_XY` — Home X and Y, then step away

### What it does

1. Runs `G28 X Y`.
2. Switches to relative positioning (`G91`) and moves **+10 mm in X and +10 mm in Y** (`custom_home_xy_re_position`) at 20 mm/s (`F1200`), waits for the move to end (`M400`).
3. Switches back to **absolute positioning (`G90`)**.

Result: X ≈ −18, Y ≈ 4.5 (the stops are at −28 / −5.5).

### Syntax

```
HOME_XY
```

### Parameters

None.

### Examples

```
HOME_XY
```
Used internally by `G28` when Z is requested and X/Y are not homed.

### Notes and cautions

- It **always leaves the printer in absolute mode (`G90`)**, even if you were in relative mode before. The extruder mode (`M82`/`M83`) is not changed.
- The step distance comes from `custom_home_xy_re_position` in `printer.cfg` (stock 10, 10).
- Z is not touched and is **not lifted** (only `G28` does the lift).

---

## `HOME_Y` — Home Y, then step away

### What it does

1. Runs `G28 Y`.
2. `G91`, then `G1 F2000 Y15`: moves **+15 mm in Y** at 33 mm/s.
3. `M400` (wait), then `G90` (absolute mode).

### Syntax

```
HOME_Y
```

### Parameters

None.

### Examples

```
HOME_Y
```
Re-home just the Y axis (for instance after a Y skip), leaving the bed 15 mm away from the stop.

### Notes and cautions

- The 15 mm and the speed are fixed, not configurable.
- Like `HOME_XY` it ends in `G90`.
- It goes through `G28`, so the 15 mm Z lift happens first.

---

*End of chapter 12. Next chapter in the command table: **Input shaper / resonance** (`ACCELEROMETER_DEBUG_READ/WRITE`, `ACCELEROMETER_MEASURE`, `ACCELEROMETER_QUERY`, `MEASURE_AXES_NOISE`, `SET_INPUT_SHAPER`, `SHAPER_CALIBRATE`, `SPECIALIZED_TEST_RESONANCES`, `TEST_RESONANCES`).*


---

# Chapter 13 — Input shaper / resonance (`ACCELEROMETER_*`, `MEASURE_AXES_NOISE`, `SET_INPUT_SHAPER`, `SHAPER_CALIBRATE`, `SPECIALIZED_TEST_RESONANCES`, `TEST_RESONANCES`)

**Input shaping** is a filter applied to the motion so the printer does not "ring" (ghosting / ripples behind sharp corners) after fast direction changes. It needs two numbers per axis: a **shaper type** and the **resonance frequency** of that axis. To find the frequency, the printer shakes itself at increasing frequencies while two small **accelerometers** record the vibration; the firmware then computes which shaper/frequency works best.

### What your printer has (stock `printer.cfg`)

| Item | Stock value |
|---|---|
| Accelerometer on the toolhead board (section `[lis2dw12]`) | measures the **X** axis (`accel_chip_x`); sample rate 400 Hz |
| Accelerometer under the bed (section `[lis2dw12 bed]`) | measures the **Y** axis (`accel_chip_y`); sample rate 400 Hz |
| Test position (`probe_points`) | X 215, Y 215, Z 20 (centre of the bed, nozzle 20 mm up) |
| Test frequency range (`min_freq` – `max_freq`) | 10 – 100 Hz |
| Test strength (`accel_per_hz`) | 150 (mm/s² of shaking per Hz) |
| Sweep speed (`hz_per_sec`) | 4 Hz per second |
| Belt-check reference (`tension_psd_x/y`, `tension_freq_x/y`) | 50000 / 40 Hz (X), 20000 / 15 Hz (Y) |
| Input shaper in use (`[input_shaper]`) | X: `mzv` at 23.0 Hz; Y: `3hump_ei` at 28.0 Hz |
| Config keys not in the stock file (firmware defaults apply) | `move_speed` 50 mm/s, `sweeping_accel` 400, `sweeping_period` 1.2, `damping_ratio_x/y` 0.1, `max_smoothing` unset |

Test results are written to `/tmp/` on the printer (a RAM disk: files vanish at reboot; you need SSH or similar to fetch them — they are not visible in Mainsail's file list).

| Command | In one line |
|---|---|
| `ACCELEROMETER_MEASURE` | Start / stop recording one accelerometer; on stop, saves a CSV of raw data. |
| `ACCELEROMETER_QUERY` | Prints one current accelerometer reading (x, y, z). |
| `ACCELEROMETER_DEBUG_READ` | Reads one register of the accelerometer chip (debug). |
| `ACCELEROMETER_DEBUG_WRITE` | Writes one register of the accelerometer chip (debug, risky). |
| `MEASURE_AXES_NOISE` | Records the accelerometers while the printer is idle and prints their noise level. |
| `TEST_RESONANCES` | Shakes the head/bed along one axis, records it, saves a CSV with the resonance spectrum. |
| `SPECIALIZED_TEST_RESONANCES` | Shakes the printer in a narrow fixed band (70–80 Hz stock) without recording anything. |
| `SHAPER_CALIBRATE` | Full automatic calibration of X and/or Y: test → best shaper → staged in the config; optional belt-tension check. |
| `SET_INPUT_SHAPER` | Changes the shaper type / frequency / damping live (not saved). |

> **Safety:** the resonance tests shake the printer **hard** (loud, strong vibration, up to several thousand mm/s² of acceleration). Never run them during a print, and make sure nothing is on the bed that could fall. None of this was test-run on a printer.

---

## `ACCELEROMETER_MEASURE` — Start / stop recording

### What it does

It is a **toggle**, per accelerometer:

1. **First call:** starts recording in the background at the chip's sample rate (400 Hz). Nothing is printed on the console (the firmware only writes a line to its internal log).
2. **Second call (same `CHIP`):** stops the recording and writes every sample to a CSV file in `/tmp/`, then prints `Writing raw accelerometer data to <file> file`.

The file name is:
- nozzle accelerometer: `/tmp/lis2dw12-<NAME>.csv`
- bed accelerometer: `/tmp/lis2dw12-bed-<NAME>.csv`

`NAME` is only read on the **second** (stopping) call.

### Syntax

```
ACCELEROMETER_MEASURE CHIP=<chip> [NAME=<text>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `CHIP` | Which accelerometer. It is the **name of its config section**. | `lis2dw12` for the toolhead accelerometer. The bed one is registered under the full section name `lis2dw12 bed` (contains a space — whether the console accepts it, e.g. in quotes, is **(to verify)**). | none — **required** |
| `NAME` | Suffix of the output file name. | Letters, digits, `-` and `_` only; anything else → error `Invalid NAME parameter`. | A date/time string. The firmware's time format is malformed, so the default is a strange digit string, not a clean `YYYYMMDD_HHMMSS`, and may repeat → always give your own `NAME`. |

### Examples

```
ACCELEROMETER_MEASURE CHIP=lis2dw12
```
Start recording on the toolhead accelerometer.

```
ACCELEROMETER_MEASURE CHIP=lis2dw12 NAME=hand_tap
```
Stop it and save `/tmp/lis2dw12-hand_tap.csv`.

### Notes and cautions

- Always stop a recording you started; a forgotten one keeps collecting data. Recording while moving is fine (the printer keeps working).
- Missing `CHIP` gives `Error on 'ACCELEROMETER_MEASURE': missing CHIP`; an unknown value gives `The value … is not valid for CHIP` (wording read from the firmware's generic multiplexed-command handler).

---

## `ACCELEROMETER_QUERY` — Read one value

### What it does

1. Starts a short measurement on the chosen accelerometer.
2. **Waits 1 second** (the command is queued with the moves, so it also waits for earlier moves to finish).
3. Stops, takes the **last** sample and prints:
   `accelerometer values (x, y, z): 0.123456, -0.234567, 9.812345` (6 decimals).
   If no sample was received: error `No accelerometer measurements found`.

The axes are already re-mapped to printer axes by the `axes_map` option of the config (stock `y,x,z`). The unit/scale was **not determined** (the scale factor is hidden in the chip's axis-mapping data); do not assume m/s² or g. Compare readings between axes and with the chip at rest rather than relying on absolute numbers.

### Syntax

```
ACCELEROMETER_QUERY CHIP=<chip>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `CHIP` | Which accelerometer (config section name, see `ACCELEROMETER_MEASURE`). | `lis2dw12` (toolhead) | none — **required** |

### Example

```
ACCELEROMETER_QUERY CHIP=lis2dw12
```
Quick sanity check that the toolhead accelerometer works: one axis should read about 9.8, the other two close to 0.

---

## `ACCELEROMETER_DEBUG_READ` — Read a chip register

### What it does

Reads register `REG` from the accelerometer chip over its SPI bus and prints `Accelerometer REG[0x<reg>] = 0x<value>` (both in hexadecimal).

### Syntax

```
ACCELEROMETER_DEBUG_READ CHIP=<chip> REG=<register>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `CHIP` | Which accelerometer. | as above | **required** |
| `REG` | Register address. | A number; hexadecimal with `0x` prefix or decimal. Valid addresses are those of the LIS2DW12 chip (e.g. `0x0F` = WHO_AM_I). No range check is done by the firmware (the value is just converted to a number; text that is not a number gives an error). | **required** |

### Example

```
ACCELEROMETER_DEBUG_READ CHIP=lis2dw12 REG=0x0F
```
Reads the chip identification register (a LIS2DW12 answers `0x44`, per its datasheet — not tested here).

### Notes

Developer/debug command; harmless (read-only).

---

## `ACCELEROMETER_DEBUG_WRITE` — Write a chip register

### What it does

Writes the value `VAL` into register `REG` of the accelerometer chip. **No message is printed.**

### Syntax

```
ACCELEROMETER_DEBUG_WRITE CHIP=<chip> REG=<register> VAL=<value>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `CHIP` | Which accelerometer. | as above | **required** |
| `REG` | Register address. | number (`0x..` or decimal) | **required** |
| `VAL` | Value to write. | number (`0x..` or decimal), one byte | **required** |

### Notes and cautions

- **Risky:** a wrong value can change the chip's range, rate or power mode and make all later measurements (and calibrations) wrong or fail until the firmware is restarted / the printer power-cycled. Use only if you know the chip's register map.

---

## `MEASURE_AXES_NOISE` — Noise level of the accelerometers

### What it does

1. Starts recording on the enabled accelerometers (the X one and the Y one from `[resonance_tester]`).
2. Waits `MEAS_TIME` seconds **without moving anything**.
3. Stops, analyses the data and prints one line per accelerometer/axis, in the form
   `Axes noise for <name>-axis accelerometer: <value> (x)` — a single noise figure per line (the trailing `(x)` is literal text in the firmware's message; which values are averaged was not decoded).
4. If an accelerometer returned nothing: error `<axis>-axis accelerometer measured no data`.

Useful to check that the sensors are healthy: a low, similar value for all is good; a very high value means vibration (fans, someone touching the printer) or a bad connection.

### Syntax

```
MEASURE_AXES_NOISE [MEAS_TIME=<seconds>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `MEAS_TIME` | Recording duration. | seconds (decimal allowed) | **2.0** |

### Example

```
MEASURE_AXES_NOISE MEAS_TIME=5
```
Measure noise for 5 s on an idle, undisturbed printer.

---

## `TEST_RESONANCES` — Resonance test of one axis

### What it does

1. Reads the options, checks `AXIS`, `OUTPUT` and `NAME`.
2. Chooses the accelerometers: by default the one configured for that axis (X → toolhead accelerometer, Y → bed accelerometer), or those named in `CHIPS` (see the caution).
3. Moves the toolhead to the **test point** (`POINT`, or the first `probe_points` entry: 215, 215, 20) at `move_speed` (50 mm/s default) and prints `Probing point (x, y, z)`. The axes **must already be homed** (this command does not home; otherwise the move is refused; the firmware contains the message `Must home axis first`, but I did not trace that it is the one raised here).
4. Prints `Testing axis <axis>`, then **vibrates** the head (X) or bed (Y) along that axis while recording:
   - The frequency is swept from `FREQ_START` to `FREQ_END` at `HZ_PER_SEC` Hz per second, with a shaking acceleration proportional to the frequency (`ACCEL_PER_HZ` × frequency). Stock: 10 → 100 Hz at 4 Hz/s ≈ **22 s** per axis.
   - On top of the vibration the head also moves slowly back and forth along the axis (**sweeping**, acceleration `SWEEPING_ACCEL`, period `SWEEPING_PERIOD`), so the measurement is not taken at a single spot. `SWEEPING_PERIOD=0` switches it off (pure vibration at the test point).
   - During the test the acceleration limit of the toolhead is temporarily set to the strongest test acceleration (and the "minimum cruise ratio" is requested as 0), and **input shaping is switched off** unless you pass `INPUT_SHAPING=1`. Everything is restored afterwards. **Two limits of this firmware apply (see `SET_VELOCITY_LIMIT`, chapter 28):** the acceleration can never exceed the configuration value (10 000 mm/s² in stock), so a test acceleration above that is cut to it, and a cruise ratio of `0` is ignored by the command, so the ratio stays as it was. Also, that `SET_VELOCITY_LIMIT` call names only the acceleration (and ratio), so every other speed/acceleration limit is reset to its configuration value during the test (to verify; not tested on a printer).
5. Analyses the recording (frequency spectrum) and writes the results to files in `/tmp/` (see `OUTPUT`). The "data written" messages go to the firmware log only, not to the console.

### Syntax

```
TEST_RESONANCES AXIS=<x|y|dx,dy> [CHIPS=<names>] [POINT=<x,y,z>]
                [OUTPUT=<resonances|raw_data|both, comma separated>] [NAME=<text>]
                [FREQ_START=<Hz>] [FREQ_END=<Hz>] [HZ_PER_SEC=<Hz/s>] [ACCEL_PER_HZ=<n>]
                [SWEEPING_ACCEL=<mm/s²>] [SWEEPING_PERIOD=<s>] [INPUT_SHAPING=<0|1>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `AXIS` | Axis to shake. | `X` or `Y` (case-insensitive); or a direction `dx,dy` such as `1,-1` for a diagonal. Errors: `Unable to parse axis direction '…'`, `Invalid format of axis '…'`. | none — **required** |
| `CHIPS` | Accelerometer(s) to record, comma separated. | **Caution:** the firmware prepends `adxl345 ` to any name that does not contain `adxl345` and looks the object up; no such object exists on this printer, so `CHIPS=lis2dw12` very probably fails. **(to verify)** — simply omit `CHIPS`. | the chip configured for the axis |
| `POINT` | Where to test. | `x,y,z` in mm. Wrong format → `Invalid POINT parameter, must be 'x,y,z' where x, y and z are valid floating point numbers`. | first `probe_points` (215,215,20) |
| `OUTPUT` | What to save. | `resonances` (spectrum), `raw_data` (every sample), or both separated by a comma. Anything else → `Unsupported output '…', only 'resonances' and 'raw_data' are supported`. | `resonances` |
| `NAME` | Suffix of the file names. | letters, digits, `-`, `_` (else `Invalid NAME parameter`) | date/time-like string (see the note at `ACCELEROMETER_MEASURE`) |
| `FREQ_START` | First frequency. | Hz, ≥ 1 | config `min_freq` (stock 10; firmware default 5) |
| `FREQ_END` | Last frequency. | Hz, ≥ `FREQ_START` and ≤ 300 | config `max_freq` (stock 100; firmware default 135) |
| `HZ_PER_SEC` | Sweep speed. | Hz per second, from 0 to 10 | config `hz_per_sec` (stock 4; firmware default 1) |
| `ACCEL_PER_HZ` | Shaking strength. | mm/s² per Hz, > 0 | config `accel_per_hz` (stock 150; firmware default 60) |
| `SWEEPING_ACCEL` | Acceleration of the slow back-and-forth movement. | mm/s², > 0 | config `sweeping_accel` (not in stock file → 400) |
| `SWEEPING_PERIOD` | Period of that movement. | seconds, ≥ 0 | config `sweeping_period` (not in stock file → 1.2) |
| `INPUT_SHAPING` | Keep the current input shaper active during the test. | 0 = switch off during the test, 1 = keep | **0** |

### Output files

- `OUTPUT=resonances` → `/tmp/resonances_<axis>_<NAME>.csv`: a header `freq,psd` then one line per frequency step (`<Hz with 1 decimal>,<power spectral density in scientific notation>`) — a **single** PSD column (not Klipper's separate x/y/z columns). If `POINT` was given, its coordinates are appended to the name.
- `OUTPUT=raw_data` → `/tmp/raw_data_<axis>_<NAME>.csv`: raw accelerometer samples.

File-name composition (axis, name, optional point and chip name with spaces turned into `_`) was read from the firmware; the exact order of the optional parts was not checked.

### Examples

```
TEST_RESONANCES AXIS=X
```
Standard X test, spectrum saved to `/tmp/resonances_x_<date>.csv`.

```
TEST_RESONANCES AXIS=Y NAME=before_belt OUTPUT=resonances,raw_data
```
Y test, saving both the spectrum and the raw samples with a name you can recognise.

```
TEST_RESONANCES AXIS=X POINT=100,100,20 FREQ_START=20 FREQ_END=80
```
Test X at another place over a narrower band (shorter test).

### Notes and cautions

- **Loud and violent.** At stock settings the shaking acceleration reaches roughly `accel_per_hz × max frequency` = 150 × 100 = 15 000 mm/s² at the top of the sweep (the firmware's test generator uses Klipper's formula) — above the 10 000 mm/s² printing limit (the toolhead's acceleration limit is capped at 10 000 during the test, chapter 28, so the real peak may be lower than the formula says; to verify). Reduce `ACCEL_PER_HZ` if it sounds or feels alarming.
- The command does **not** change your input shaper; it only produces data. To get a recommendation use `SHAPER_CALIBRATE`.
- The test point is Z = 20 above the bed: remove anything tall from the bed; the nozzle should be cold or at least not touching anything.
- Must be homed first (use `G28`).

---

## `SPECIALIZED_TEST_RESONANCES` — Shake in a fixed narrow band, no recording

### What it does

Same as `TEST_RESONANCES` as far as moving to the test point and shaking is concerned, **but**:

1. **No accelerometer is used and no file is written.** It only shakes the printer.
2. `FREQ_START` and `FREQ_END` are **forced** to the config values `test_freq_start` and `test_freq_end` (stock **70** and **80** Hz); anything you pass for them is overwritten.
3. The other settings (`ACCEL_PER_HZ`, `HZ_PER_SEC`, `SWEEPING_*`, `INPUT_SHAPING`, `POINT`) work as in `TEST_RESONANCES`.

Likely purpose: exciting the machine at the frequencies of the belt-tension "check" while you measure the sound or vibration with an external tool **(purpose inferred from the config names — to verify)**. Stock test length: (80 − 70) / 4 Hz/s ≈ 2.5 s.

### Syntax

```
SPECIALIZED_TEST_RESONANCES AXIS=<x|y|dx,dy> [POINT=<x,y,z>]
                            [ACCEL_PER_HZ=<n>] [HZ_PER_SEC=<Hz/s>]
                            [SWEEPING_ACCEL=<mm/s²>] [SWEEPING_PERIOD=<s>] [INPUT_SHAPING=<0|1>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `AXIS` | Axis to shake. | as `TEST_RESONANCES` | **required** |
| `POINT` | Test position. | `x,y,z` | first `probe_points` |
| `FREQ_START`, `FREQ_END` | (accepted, but **ignored** — replaced by `test_freq_start` / `test_freq_end`) | — | 70 / 80 Hz stock |
| `ACCEL_PER_HZ`, `HZ_PER_SEC`, `SWEEPING_ACCEL`, `SWEEPING_PERIOD`, `INPUT_SHAPING` | As in `TEST_RESONANCES`. | | same defaults |

### Example

```
SPECIALIZED_TEST_RESONANCES AXIS=Y
```
Shake the bed (Y) in the 70–80 Hz band.

### Notes

Same safety remarks as `TEST_RESONANCES`; printer must be homed.

---

## `SHAPER_CALIBRATE` — Automatic input-shaper calibration

### What it does

1. **Homing if needed:** if X, Y and Z are not all homed, it waits 1 s and runs `G28 W` (home all axes, Z at the LeviQ3 platform spot — see chapter 12). If already homed, nothing is homed.
2. Chooses the axes: `AXIS=X`, `AXIS=Y`, or **both** (X then Y) if `AXIS` is omitted. Anything else → error `Unsupported axis '…'`.
3. For each axis it runs **the same shaking test as `TEST_RESONANCES`** (same moves, same parameters, same test point from `probe_points`; no `POINT`, `CHIPS` or `OUTPUT` options here).
4. Computes the **best shaper** among the standard Klipper types (`zv`, `mzv`, `zvd`, `ei`, `2hump_ei`, `3hump_ei`) and its frequency, respecting `MAX_SMOOTHING` if set. The recommendation `Recommended shaper_type_x = … , shaper_freq_x = … Hz` is written **only to the firmware log, not shown on the console** (log file `/tmp/gklib.log`; Mainsail's console will show little).
5. If `IS_SAVE=1` (default): **stages** `shaper_type_<axis>` and `shaper_freq_<axis>` (frequency with 1 decimal) in the `[input_shaper]` section of the pending configuration. **Nothing is applied yet and nothing is on disk until you run `SAVE_CONFIG`** (which in this firmware reloads the configuration in place without restarting; see chapter 14) (verified: the firmware only stores the two values in its pending-config list; nothing is applied until `SAVE_CONFIG`).
6. Writes the spectrum + shaper curves to `/tmp/calibration_data_<axis>_<NAME>.csv` (log message only).
7. If `BELT_CHECK_ENABLE=1` (default): **belt check** — compares the measured main resonance with the reference values `tension_psd_*` / `tension_freq_*` of the config (only if both are > 0; stock yes). It prints one of:
   - `axis x belt break` — measured peak height is below **1 %** of the reference height (belt looks broken / disconnected),
   - `axis x belt loose` — peak height below the reference height, **or** peak frequency below the reference frequency,
   - nothing if the belt looks fine.
   (Verified: the compared values are the height (`max_psd`) and frequency (`max_freq`) of the strongest peak of the best-shaper fit, which the firmware also writes to its log.)
8. A cancel from the touchscreen/web interface during the test aborts with `Instruction cancellation`.

### Syntax

```
SHAPER_CALIBRATE [AXIS=<X|Y>] [NAME=<text>] [MAX_SMOOTHING=<value>]
                 [IS_SAVE=<0|1>] [BELT_CHECK_ENABLE=<0|1>] [INPUT_SHAPING=<0|1>]
                 [FREQ_START=<Hz>] [FREQ_END=<Hz>] [HZ_PER_SEC=<Hz/s>] [ACCEL_PER_HZ=<n>]
                 [SWEEPING_ACCEL=<mm/s²>] [SWEEPING_PERIOD=<s>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `AXIS` | Axis to calibrate. | `X`, `Y` | both |
| `NAME` | Suffix of the CSV file name. | letters, digits, `-`, `_` | date/time-like string |
| `MAX_SMOOTHING` | Upper limit of the smoothing (print-detail blur) accepted when choosing the shaper; lower value = gentler shapers only. | number ≥ 0.05 (Klipper suggests 0.2) | config `max_smoothing`; unset in the stock file → no limit |
| `IS_SAVE` | Stage the result in the config. | 1 = yes, 0 = only compute | **1** |
| `BELT_CHECK_ENABLE` | Do the belt check. | 1 = yes, 0 = no | **1** |
| `INPUT_SHAPING` | Keep current shaper active during the test (rarely useful: it distorts the measurement). | 0 / 1 | **0** |
| `FREQ_START`, `FREQ_END`, `HZ_PER_SEC`, `ACCEL_PER_HZ`, `SWEEPING_ACCEL`, `SWEEPING_PERIOD` | Test settings, as in `TEST_RESONANCES`. | | config values |

### Examples

```
SHAPER_CALIBRATE
```
Calibrate X and Y (about 22 s each plus analysis), stage the best shaper settings.

```
SHAPER_CALIBRATE AXIS=Y IS_SAVE=0 BELT_CHECK_ENABLE=0
```
Only compute Y and do not touch the config or check belts.

### Notes and cautions

- **Heavy shaking**, see the safety note at the top of the chapter. Do not run during a print; the nozzle will move to the bed centre at Z 20.
- Because the recommendation only appears in the firmware log (`/tmp/gklib.log`), the easiest way to see it is to run `SAVE_CONFIG` and read the new `[input_shaper]` values at the end of `printer.cfg`, or to read the log over SSH. If you prefer to see the numbers yourself, use `TEST_RESONANCES` and analyse the CSV.
- `SAVE_CONFIG` writes the file (with a timestamped backup); it does not restart the firmware in this version (it would also save any other staged changes, e.g. a bed mesh).
- The default `NAME` is built with a faulty time format, so file names may collide between runs — pass your own `NAME` if you want to keep results.

---

## `SET_INPUT_SHAPER` — Change the shaper live

### What it does

1. For X and Y, reads any given parameter and updates that axis's settings.
2. If anything changed, flushes the step generation and reconfigures the steppers' motion with the new shaper (a brief hiccup of the motion queue; **avoid doing it mid-print**). If the shaper cannot be built: error `Failed to configure shaper(s) … with given parameters, error: …`.
3. Prints the resulting settings for each axis on one line, alphabetically: e.g. `damping_ratio_x:0.100000 shaper_freq_x:23.000 shaper_type_x:mzv` (then the same for `y`).

A frequency of **0 disables shaping** on that axis. The change is **temporary**: it is lost on restart (the `[input_shaper]` section of `printer.cfg`, or a `SAVE_CONFIG` after `SHAPER_CALIBRATE`, is what is loaded at start).

### Syntax

```
SET_INPUT_SHAPER [SHAPER_FREQ_X=<Hz>] [SHAPER_FREQ_Y=<Hz>]
                 [DAMPING_RATIO_X=<0-1>] [DAMPING_RATIO_Y=<0-1>]
                 [SHAPER_TYPE=<type>] [SHAPER_TYPE_X=<type>] [SHAPER_TYPE_Y=<type>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `SHAPER_FREQ_X`, `SHAPER_FREQ_Y` | Resonance frequency the shaper cancels. | Hz, ≥ 0; **0 = no shaping** | current value |
| `DAMPING_RATIO_X`, `DAMPING_RATIO_Y` | Damping of the resonance (used by `zv`, `mzv`, `zvd`, `ei` families). | 0 … 1 | current value (config default 0.1) |
| `SHAPER_TYPE` | Shaper type for **both** axes. | `zv`, `mzv`, `zvd`, `ei`, `2hump_ei`, `3hump_ei` (case-insensitive); else `Unsupported shaper type: …` | current type |
| `SHAPER_TYPE_X`, `SHAPER_TYPE_Y` | Shaper type for one axis; overrides `SHAPER_TYPE`. | same list | `SHAPER_TYPE` if given, else current |

With no parameter at all it just prints the current settings.

### Examples

```
SET_INPUT_SHAPER
```
Show the current shaper settings.

```
SET_INPUT_SHAPER SHAPER_FREQ_X=0 SHAPER_FREQ_Y=0
```
Disable input shaping on both axes (e.g. to compare print quality).

```
SET_INPUT_SHAPER SHAPER_TYPE_X=mzv SHAPER_FREQ_X=45 SHAPER_TYPE_Y=zvd SHAPER_FREQ_Y=38.5
```
Set both axes.

### Notes and cautions

- Stock values are X `mzv` 23 Hz and Y `3hump_ei` 28 Hz. `3hump_ei`/`2hump_ei`/`ei` smooth corners more than `mzv`/`zv`.
- Not persistent; change `[input_shaper]` in `printer.cfg` (or `SAVE_CONFIG` after a calibration) to keep it.
- Changing it while printing flushes the motion pipeline: do it between prints.

---

*End of chapter 13.*


---

# Chapter 14 — LeviQ3 auto-leveling (`LEVIQ`, `LEVIQ_PREHEATING`, `LEVIQ_WIPING`, `LEVIQ_PROBE`, `LEVIQ_SET_ZERO`, `LEVIQ_AUTO_ZOFFSET`, `LEVIQ_TEMP_OFFSET`, `LEVIQ_AUTO_ZOFFSET_ON_OFF`)

**LeviQ 3** is Anycubic's "no-touch-screen-fuss" leveling: the nozzle itself is the sensor (a strain gauge in the toolhead measures how hard the nozzle presses on the bed), so the bed can be measured with a **hot-but-cool-enough, cleaned nozzle** and the Z offset can be found automatically. These commands are the building blocks the touchscreen "auto leveling" button uses; `G9111` (chapter 2) calls two of them (`LEVIQ_AUTO_ZOFFSET`, `LEVIQ_TEMP_OFFSET`).

| Command | In one line |
|---|---|
| `LEVIQ` | The complete leveling run: preheat → wipe → measure the bed mesh. What the touchscreen runs. |
| `LEVIQ_PREHEATING` | Step 1 of `LEVIQ`: heats nozzle and bed, parks the toolhead, retracts a little. |
| `LEVIQ_WIPING` | Step 2: cleans the nozzle (silicone wiper + "fine wipe" pad), cools to probing temperature, re-homes Z. |
| `LEVIQ_PROBE` | Step 3: measures an adaptive bed mesh (`BED_MESH_CALIBRATE ADAPTIVE=1`) and lifts the head. |
| `LEVIQ_SET_ZERO` | Resets the Z offsets in memory to 0 (the starting point for a measurement). |
| `LEVIQ_AUTO_ZOFFSET` | Measures the nozzle-to-bed Z offset with the strain gauge and corrects it. |
| `LEVIQ_TEMP_OFFSET` | Adds a small Z correction for the nozzle's printing temperature (hot nozzles are longer). |
| `LEVIQ_AUTO_ZOFFSET_ON_OFF` | Switches the automatic Z-offset measurement on or off (saved in the config). |

### Settings used (section `[leviQ3]` of `printer.cfg`)

| Setting | Stock value | Firmware default if missing | Used for |
|---|---|---|---|
| `bed_temp` | 60 | 60 | Bed temperature for preheating/wiping. |
| `extru_temp` | 170 | 190 | Nozzle temperature while wiping. |
| `extru_end_temp` | 140 | 140 | Nozzle temperature while probing (low, so it does not ooze). |
| `to_z_up_speed` | 15 mm/s | 15 | Speed of Z lifts. |
| `to_zero_speed` | 200 mm/s | 50 | Speed of travel moves during wiping. |
| `z_up` | 30 mm | 30 | Height the head is lifted to after probing / before the fine wipe. |
| `wipe_fine_pos` | 30, 420 | required | X, Y of the "fine wipe" spot (first value 0 or missing = no fine wipe). |
| `platform_offset_extru` | 0, 440, 0.2 | required | X, Y of the parking spot used by `LEVIQ_PREHEATING`. |
| `platform_offset` | 30, 414, 0.2 | required | Spot where Z is homed (`G28 W` uses X, Y−10, see chapter 12). |
| `expansion_factor` | 0.06 | 0.0375 | Thermal Z correction (mm) per 80 °C above 140 °C — `LEVIQ_TEMP_OFFSET`. |
| `auto_step` | 0.03 | 0.03 | Height (mm) of one lift step of the Z-offset routine. Two measurements "agree" if their results differ by at most 2 × `auto_step` + 0.0001 mm. |
| `zoffset_factor` | 0.03 | 0.03 | Width (mm) of the "suspiciously small result" window that triggers a clean-and-measure-again (see `LEVIQ_AUTO_ZOFFSET`). |
| `return_distance` | not set | 0.075 | Distance (mm) of the final move back after the sensor reading has settled. |
| `noise_diff` | not set | 2500 | Largest change between two sensor readings that still counts as "no change". |
| `max_diff` | not set | 5000 | Read, but only written to the firmware log by the Z-offset routine. |
| `auto_zoffset_on_off` | not set | 1 (on) | Whether the automatic Z-offset measurement is enabled. |

The other keys of the stock section (`extru_out`, `extru_in`, `vibrations_*`, `direction`, `threshold_factor`, …) are not used by these commands.

> **General warnings.** These commands heat, move and home the printer on their own. Run them only with an empty, clean bed and **never during a print**. Errors in the multi-step commands are mostly **only written to the firmware log** — the console may stay silent while the sequence quietly stops. None of this was test-run on a printer.

---

## `LEVIQ_SET_ZERO` — Reset the Z offsets to zero

### What it does

Sets to **0** (in memory only):
1. the probe's Z offset,
2. an internal probe/toolhead offset value,
3. the Z part of the G-code offset (what `SET_GCODE_OFFSET Z=…` / baby-stepping would have set) and of the stored "homing" position.

It moves nothing and prints nothing. Nothing is written to `printer.cfg`. The firmware's help text ("Calculates the z_offset automatically") is misleading: it only resets. It is called at the start of `LEVIQ_PREHEATING` and of every measurement of `LEVIQ_AUTO_ZOFFSET`.

### Syntax

```
LEVIQ_SET_ZERO
```

### Parameters

None.

### Notes and cautions

- Any live Z baby-step adjustment is lost. The saved `z_offset` in `printer.cfg` is untouched but is not in effect until it is applied again (a restart, or the commands below).
- **`LEVIQ_SET_ZERO` followed by `SAVE_CONFIG` does not zero the saved offset.** `LEVIQ_SET_ZERO` changes live values only and stages nothing, and `SAVE_CONFIG` writes only staged items. Observed on a printer: after `SET_GCODE_OFFSET Z=0 MOVE=1`, `LEVIQ_SET_ZERO` and `SAVE_CONFIG`, the live G-code offset (`gcode_move.homing_origin`) was 0, but the saved file (`printer_mutable.cfg`) still held the earlier `probe` `z_offset` (−0.0150). Until the next restart the printer runs with a probe offset of 0, which is not a usable first-layer height. To get back to a sensible value, run `RESTART` (reloads the saved value), or measure again with `LEVIQ_AUTO_ZOFFSET IS_SAVE=1` and `SAVE_CONFIG`.

---

## `LEVIQ_PREHEATING` — Step 1: heat and park

### What it does

1. Starts from the defaults **bed 60 °C, nozzle 170 °C** (fixed values, whatever `bed_temp`/`extru_temp` say), then applies the parameters `B` (bed) and `S` (nozzle) if given. `E` (probing temperature, default `extru_end_temp`) is also accepted.
2. Computes the bed temperature to use: on the **Kobra 3 Max** (this printer), if the bed temperature is above 0 it is raised by **10 °C** (limited to the bed's `max_safe_temp`: 100 on the stock config, 90 if unset); on the smaller Kobra 3 models it is only limited to the maximum.
3. **If you passed `B` (Kobra 3 Max / Max V2 only):** heats the bed to the temperature computed in step 2 (so `B` + 10 °C, capped) and waits (`M190`), then waits **120 seconds** (heat soak). Without `B` this step is skipped. *(Verified in the firmware: the test is simply whether a `B` parameter is present in the command, whatever its value.)*
4. Runs `LEVIQ_SET_ZERO`.
5. Sets the nozzle target (`M104`, no waiting). Then the bed: on the Kobra 3 / 3 V2 it **waits** for the bed temperature; on the Kobra 3 Max it only **sets** it (no waiting — `LEVIQ_WIPING` waits later).
6. Heats the nozzle and **waits** (`M109`).
7. Resets the extruder position and **retracts 10 mm** (`G92 E0`, `G1 E-10 F400`), waits for moves.
8. **If X and Y are homed**, moves to the parking spot `platform_offset_extru` (stock X 0, Y 440) at 9600 mm/min and waits.
9. Finally makes sure the nozzle really is at the target (`M109` once more if it is below).

If a cancel is requested the command stops at the next step.

### Syntax

```
LEVIQ_PREHEATING [B=<bed °C>] [S=<nozzle °C>] [E=<probing °C>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `B` | Bed temperature. Also triggers the 120 s heat soak (step 3). | °C | 60 |
| `S` | Nozzle temperature for wiping. | °C | 170 |
| `E` | Nozzle temperature for probing (kept for the later steps). | °C | `extru_end_temp` (140) |

Note the unusual one-letter names, and that the firmware's help lists **no** parameters (they are read directly from the command line).

### Example

```
LEVIQ_PREHEATING B=60 S=170
```
Heat for leveling with the usual temperatures; waits 2 minutes for the bed to soak first.

### Notes and cautions

- The nozzle retracts 10 mm: if filament is loaded it will be pulled back; the later wiping/probing expects this.
- The parking position (X 0, Y 440 stock) is reached only if the axes are already homed. Home first (`G28`).

---

## `LEVIQ_WIPING` — Step 2: clean the nozzle and cool to probing temperature

### What it does

1. Runs **`G9112`** (silicone-pad nozzle wipe, chapter 2) and waits. (Note `G9112` does nothing if the "already cleaned" memory is set.)
2. **Clears the bed mesh.**
3. Cools/heats the nozzle to the **probing temperature** `extru_end_temp` (140) and **waits**, then heats the bed to `bed_temp` (60) and **waits**; part-cooling fan off.
4. **Fine wipe**, only if `wipe_fine_pos` has non-zero X and Y (stock 30, 420): lifts to `z_up` (30 mm), travels to the fine-wipe spot, homes Z there (`G28 Z`), and then presses the nozzle **four times** onto the pad: at X = spot, +2, +4, +6 mm it goes to Z = 1 mm and then down to Z = −0.2 mm (speeds are half of `to_z_up_speed` / `to_zero_speed`). *(Sequence read from the code; the purpose — scraping residual ooze — is inferred.)*
5. Otherwise (no fine wipe) it only homes Z (`G28 Z`).
6. Waits for moves, pauses 1 s.

### Syntax

```
LEVIQ_WIPING
```

### Parameters

None.

### Notes and cautions

- Needs the nozzle heater to reach 140 °C: it will **wait for the nozzle to cool down** from a higher temperature, which can take minutes.
- The fine wipe presses the nozzle 0.2 mm below Z 0 at the `wipe_fine_pos` spot; if you changed the bed/plate setup, check that spot first.
- A cancel stops the sequence at the next step.

---

## `LEVIQ_PROBE` — Step 3: measure the bed mesh

### What it does

1. Runs **`BED_MESH_CALIBRATE ADAPTIVE=1`** (chapter 3). Because no print is running, the "adaptive" area is the whole bed (full mesh). The mesh is stored in profile `default` as usual (session only; see chapter 3).
2. **If this fails** (for example `probe samples exceed samples_tolerance`, or anything else): the error is logged and the command runs its **emergency clean-up**: `G9113` (reset "nozzle cleaned"), **`M190 S0`, `M109 S0`** (heaters off), `M107` (fan off) and **`M84` (motors off)**. No clear message is shown.
3. If it succeeds: waits for moves, restores the saved probe offset (the one taken before `LEVIQ_SET_ZERO`) to the probe, turns the part fan off, **lifts the head to `z_up`** (30 mm) at `to_z_up_speed`, and waits.

### Syntax

```
LEVIQ_PROBE
```

### Parameters

None.

### Notes and cautions

- A failed probing run **turns off your heaters and releases the motors** — do not run it with something you do not want cooled.
- The nozzle should be clean and at probing temperature (that is what steps 1–2 are for); probing with a dirty or hot nozzle gives wrong meshes.

---

## `LEVIQ` — Complete leveling run

### What it does

1. If the filament hub's object is missing, an error is raised (log only).
2. Depending on the state, runs:
   - **no ACE hub in use** (the filament hub reports `ext_spool: 1` in Moonraker, as on a printer fed from the external spool): `LEVIQ_PREHEATING`, `LEVIQ_WIPING`, then `LEVIQ_PROBE`;
   - **ACE hub in use:** if the printer is **not** in "flow calibration" mode: `LEVIQ_PREHEATING` and `LEVIQ_WIPING`; then in all cases `BED_MESH_CLEAR`, `M107`, `G28 Z`, `M400`, 1 s pause, and `LEVIQ_PROBE`. So during a flow calibration the preheating and wiping are skipped (the tail of the wiping step is still done). *(Both conditions verified in the firmware; why flow calibration skips the wiping is my interpretation: the calibration prepares the nozzle itself.)*
3. If `SYNC=enable` is given, requests a **firmware restart** when everything is done.
4. If any step fails, the failure is written to the log only and the sequence stops there (no console message).

### Syntax

```
LEVIQ [SYNC=<enable|disable>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `SYNC` | Restart the firmware at the end (used by the touchscreen to refresh its state). | `enable` = restart; anything else = no restart | `disable` |

### Examples

```
G28
LEVIQ
```
Home, then do a complete leveling run (preheat 60 °C bed / 170 °C wipe, wipe, probe).

### Notes and cautions

- **Takes many minutes** (heat soak, cooling to 140 °C, 81 probe points).
- The result is a bed mesh in the session; use `SAVE_CONFIG` to keep it (see chapter 3).
- Unlike `G9111`, `LEVIQ` does not measure the Z offset — it builds the mesh only.

---

## `LEVIQ_AUTO_ZOFFSET` — Measure and correct the Z offset

### What it does

(Used by `G9111` before every print.)

1. Heats/cools the nozzle to the probing temperature (`extru_end_temp`, 140) and **waits**.
2. Fan off; relative-extrusion mode; a tiny **0.05 mm retract** (`G1 E-0.05 F100`); back to absolute extrusion.
3. **If the automatic Z-offset feature is off** (`auto_zoffset_on_off` ≤ 0, see `LEVIQ_AUTO_ZOFFSET_ON_OFF`): it simply homes Z (`G28 Z`) and stops.
4. **If it is on:** runs one **measurement** (below) twice in a row and compares the two resulting Z offsets (step 5).
5. **Checking the two results** (read from the firmware):
   - If the two results differ by more than `2 × auto_step + 0.0001` mm (0.0601 mm stock), they are called inconsistent: the whole measurement is done **two more times**, and if those two again disagree by that much, the Z offset that was in effect before the command is put back **in memory** (the file is not rewritten by the restore). If the new pair agrees, its result stays.
   - If they agree and the second result is **not** between `−zoffset_factor` and 0 mm (stock: −0.03 … 0), the command is finished.
   - If they agree but the result **is** in that window (−0.03 … 0 mm), the routine treats it as "suspiciously small" and **cleans and measures again**: it parks the head at `platform_offset_extru` (X 0, Y 440 stock), heats the nozzle to `extru_temp` (170) and waits, runs `G9113` and `G9112` (a forced nozzle wipe, chapter 2), cools back to `extru_end_temp` (140), fan off, and repeats the double measurement with the same check as above. This clean-and-measure pass is done `RETRY_COUNT` times (at least once).
6. At the end the strain-gauge sensor is switched off again. Details are written to the firmware log (`unpressure:`, `cur_adc:`, `up:` lines).

**One measurement:**
1. Switches the strain-gauge sensor on (waits 0.5 s), resets the offsets (`LEVIQ_SET_ZERO`), homes Z (`G28 Z`), waits 1 s and takes a first sensor reading.
2. Moves to Z = 0.0 at the probe's speed, waits 0.2 s and reads the sensor again.
3. **Steps:** repeatedly applies `set_gcode_offset Z_ADJUST=+<auto_step> move=1` (a 0.03 mm lift of the nozzle: the firmware's `SET_GCODE_OFFSET` adds `Z_ADJUST` to the offset and, with `move=1`, moves Z by that same amount, and positive Z is away from the bed), waits 0.2 s and reads the sensor. It stops when **two consecutive steps** each changed the reading by no more than `noise_diff` (the reading has settled, the nozzle no longer presses on the bed), so there are at least three steps. The firmware has no upper limit on the number of steps.
4. Moves back by `return_distance` (0.075 mm) in the opposite direction. The total correction is therefore *number of steps × `auto_step` − `return_distance`* (3 steps: 0.015 mm). Because `LEVIQ_SET_ZERO` zeroed the offsets first, this total becomes the probe's new `z_offset` (with the opposite sign, as in standard Klipper).
5. If `IS_SAVE=1`: `Z_OFFSET_APPLY_PROBE` (stores the offset as the probe's `z_offset`) and `SAVE_CONFIG` (writes it to `printer.cfg`). Then it lifts to Z = 5 mm (`G1 Z5.0 F4800`).
6. If the sensor reading fails, the error is logged and the previous Z offset is put back in memory.

*What is still not certain:* the "cause" of the small-result window (my reading: with the stock values it is exactly the three-step minimum, i.e. the sensor never saw the nozzle pressing at Z 0) come from the arithmetic, not from a test on a printer. (The `Z_ADJUST` direction itself was checked in the firmware's `SET_GCODE_OFFSET` code.) The "two results" are the probe `z_offset` as updated by `Z_OFFSET_APPLY_PROBE`; with `IS_SAVE=0` that update does not happen, so the consistency checks compare the same stale value and the command is only reliable with `IS_SAVE=1`.

### Reading the log of a measurement

Each measurement writes lines like this to the firmware log (on a Rinkhals printer `/tmp/gklib.log`; filter with `grep -E "unpressure|cur_adc|up:" /tmp/gklib.log`):

```
DEBUG unpressure:-1428873 cur_adc:-1396991 up:-1401991
DEBUG unpressure:-1428873 cur_adc:-1396139 up:-1401139 diff:852
DEBUG unpressure:-1428873 cur_adc:-1402281 up:-1407281 diff:6142
...
```

- **First line of a block:** the start of one measurement. `unpressure` is the first sensor reading, taken after `G28 Z`; `cur_adc` is the reading at Z = 0.
- **Following lines:** one per lift step. `cur_adc` is the new reading, `diff` is its change from the previous reading. The `diff` of the first step is ignored by the routine.
- **`up`:** always `cur_adc` minus `max_diff` (5000 by default); it is only logged and not used for decisions.
- **Where it stops:** the measurement ends at the first two **consecutive** `diff` values of at most `noise_diff` (2500 by default), not counting the first step. The number of lines after the first one is the number of steps; the result is *steps × `auto_step` − `return_distance`* (7 steps with the stock values: 0.135 mm).
- **Healthy pattern** (observed on a Kobra 3 Max): the readings fall by roughly 4000–21000 counts per step while the nozzle is still pressing, then settle to changes of about 600–1700. A `noise_diff` of 2500 sits between those two ranges. If the plateau changes by more than `noise_diff`, the routine keeps stepping; if the slope changes by less, it stops too early.
- **Baseline drift is normal:** `unpressure` was seen to move by about 26000 counts between measurements, which is comparable to the whole pressing signal. It is not used for the result.
- **Result variation:** in four logged measurements the result was 7, 7, 7 and 6 steps (0.135, 0.135, 0.135, 0.105 mm); one step (0.03 mm) of variation is the resolution of the method. A result far from the others (for example 13 steps) shows up as many more lines with `diff` above `noise_diff` — keep that block of the log when you report or analyse it.
- **Comparing runs without saving:** with `IS_SAVE=0` nothing is written to the configuration; the result is the live G-code Z offset (`gcode_move.homing_origin`, Z) after the run, which is reset by the next run's `LEVIQ_SET_ZERO`. (Derived from the code, not tested.)

### Syntax

```
LEVIQ_AUTO_ZOFFSET [IS_SAVE=<0|1>] [RETRY_COUNT=<n>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `IS_SAVE` | Store the result: apply it to the probe and run `SAVE_CONFIG`. | 1 = yes, 0 = measure/apply for this session only | **1** |
| `RETRY_COUNT` | How many times the "clean the nozzle and measure again" pass (step 5) is done when the result is suspiciously small. | integer; 0 behaves like 1 (the pass always runs once when it is triggered) | **1** |

### Example

```
G28
LEVIQ_AUTO_ZOFFSET IS_SAVE=0
```
Measure the Z offset for this session only, without touching `printer.cfg`.

### Notes and cautions

- Needs a **clean nozzle** at 140 °C and a clean bed; a blob on the nozzle gives a wrong offset (the correction is written to `printer.cfg` with `IS_SAVE=1`).
- It also changes the saved `z_offset` on every run when saving is on.
- The command help text ("LEVIQ3 clear coefficient K") does not describe what it does.

---

## `LEVIQ_TEMP_OFFSET` — Z correction for the printing temperature

### What it does

A hot nozzle is slightly longer than a cooler one, so the Z offset measured at 140 °C is a little too far from the bed at 220 °C. This command compensates:

1. Takes the **nozzle temperature of the print job**: the number the firmware read from the G-code file when it was opened: the header comment `; nozzle_temperature = …` or `; first_layer_temperature = …` (if the line holds a list, the first number), or an `M109 S…` line. It is 0 when no print file is known. It then computes
   `correction = (temperature − 140) × expansion_factor / 80` (stock: `(T − 140) × 0.06 / 80` → 0.045 mm at 200 °C, 0.06 mm at 220 °C). If the temperature is 0 or unknown, the correction is 0; a temperature below 140 °C gives a negative correction.
2. Applies it with `set_gcode_offset Z_ADJUST=<correction> move=1` (moves the head by that amount and adjusts the G-code offset).
3. Runs `Z_OFFSET_APPLY_PROBE` and `M400`, then **`SAVE_CONFIG`** (see the note below).

*(Verified: the temperature is a value the firmware stored when the print file was opened. It is taken from the two header comments above or from a line of the file that starts with `M109 ` followed by `S<temperature>` (only if the number is above 0). A slicer that writes the temperature under another header name, e.g. `nozzle_temperature_initial_layer`, therefore works only through its `M109 S…` line. How far into the file the firmware reads, and which line wins when several match, was not checked.)*

### Syntax

```
LEVIQ_TEMP_OFFSET
```

### Parameters

None.

### Notes and cautions

- `G9111` calls this after reaching the printing temperature.
- It runs `SAVE_CONFIG` every time, so the saved configuration (on this printer `printer_mutable.cfg`) is rewritten at each print start. In this firmware the `SAVE_CONFIG` command writes the file and does **not** restart the firmware (this is why it can be used in the middle of a print start-up; see the correction note in chapter 3).

---

## `LEVIQ_AUTO_ZOFFSET_ON_OFF` — Enable / disable the automatic Z-offset

### What it does

Stores the on/off switch used by `LEVIQ_AUTO_ZOFFSET`: sets `auto_zoffset_on_off` in the `[leviQ3]` section and runs `SAVE_CONFIG`, so it survives restarts. With the feature **off**, `LEVIQ_AUTO_ZOFFSET` (and therefore `G9111`) only homes Z and does not measure.

### Syntax

```
LEVIQ_AUTO_ZOFFSET_ON_OFF ON_OFF=<0|1>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `ON_OFF` | Switch. | **1 (or any value > 0) = on**, 0 = off | the current setting (so with no parameter it just re-saves it) |

### Examples

```
LEVIQ_AUTO_ZOFFSET_ON_OFF ON_OFF=0
```
Turn the automatic Z-offset off (the `G9111` start sequence will not measure any more).

### Notes and cautions

- Even with it off, `G9111` still heats, cleans the nozzle and calls the other steps; only the measurement is skipped.
- The value is not range-checked: any integer is stored; only "> 0" means on.

---

*End of chapter 14.*


---

# Chapter 15 — Low-level motion (`M18`, `M84`, `SET_STEPPER_ENABLE`, `STEPPER_BUZZ`, `FORCE_MOVE`, `SET_KINEMATIC_POSITION`, `MANUAL_STEPPER`)

These commands act on the **individual motors (steppers)** instead of on the toolhead position. They are mostly maintenance and diagnostic tools. The stepper names on this printer come from `printer.cfg`: **`stepper_x`**, **`stepper_y`**, **`stepper_y1`** (second Y motor on the Max), **`stepper_z`**, and and the extruder motor, which is named after its config section, **`extruder`**. All motors share one enable line (`enable_pin: !PC3`). The firmware counts how many motors have asked for that line: it is physically released only when **every** motor has been disabled (see `SET_STEPPER_ENABLE`).

| Command | In one line |
|---|---|
| `M18` / `M84` | Switch **all** motors off (and forget the homed state). |
| `SET_STEPPER_ENABLE` | Switch one motor on or off by name. |
| `STEPPER_BUZZ` | Wiggle one motor 10 times to identify it. |
| `FORCE_MOVE` | Move one motor by a distance, ignoring limits and kinematics. |
| `SET_KINEMATIC_POSITION` | Tell the firmware where the toolhead is, without moving. |
| `MANUAL_STEPPER` | Drive an extra, separately configured motor (**not available with the stock config**). |

> **Safety:** `FORCE_MOVE` and `SET_KINEMATIC_POSITION` bypass all the protections (limits, homing state). Use them only when you know what you are doing, with the motors free to move, and re-home (`G28`) afterwards.

---

## `M18` / `M84` — Disable all motors

### What it does

`M18` and `M84` are the same command (same handler). It takes **no parameters** — unlike other firmwares you cannot disable only some axes (`M84 X Y` disables everything).

1. Waits 0.1 s (a short pause in the movement queue), so the moves already queued are finished first.
2. Switches **every** motor off (X, Y, Y1, Z, extruder). Held positions are lost: the carriage and bed can be moved by hand. Whether the bed sinks under its own weight when Z is released is a mechanical question the firmware cannot answer; keep clear of the bed after `M84`.
3. Tells the kinematics that the motors are off: the **homed state of X, Y and Z is erased** (`homed_axes` becomes empty). The next move in a print needs a new `G28`.
4. Sends the internal "motors off" notification and waits another 0.1 s.

### Syntax

```
M18
M84
```

### Parameters

None.

### Examples

```
M84
```
Typical end of a print: motors off, axes un-homed.

### Notes and cautions

- Several end-of-print G-codes use `M84`; remember it **un-homes the printer**, so do not send moves afterwards without `G28`.
- Motors also turn off by themselves after the idle timeout (see `SET_IDLE_TIMEOUT`, chapter 17).
- A move command re-enables the motors automatically.

---

## `SET_STEPPER_ENABLE` — Switch one motor on or off

### What it does

1. Looks up the stepper called `STEPPER`. If the name is unknown: error **`set_stepper_enable: Invalid stepper <name>`**.
2. Waits for the queued moves (dwell), then enables or disables the motor and logs "has been manually enabled/disabled" (log file only, nothing on the console).
3. Waits again before returning.

### Syntax

```
SET_STEPPER_ENABLE STEPPER=<name> [ENABLE=<0|1>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Motor name (**required**). | `stepper_x`, `stepper_y`, `stepper_y1`, `stepper_z`, extruder (see chapter intro) | – |
| `ENABLE` | Target state. | 1 = on, 0 = off | **1** |

### Examples

```
SET_STEPPER_ENABLE STEPPER=stepper_z ENABLE=0
```
Release the Z motor, e.g. to move the bed by hand.

```
SET_STEPPER_ENABLE STEPPER=stepper_x ENABLE=1
```
Energise the X motor again.

### Notes and cautions

- **Shared enable line (verified in the firmware):** all steppers use the same enable pin and the firmware keeps a counter of enabled motors on it. `ENABLE=0` on one motor only lowers the counter; the motors are physically released **when the last enabled motor is disabled**. So disabling `stepper_z` alone leaves the line energised as long as X/Y/Y1/extruder are still enabled, and the motor stays held. To really free Z, disable all of them (`M84`).
- Unlike `M18`, this command does **not** erase the homed state. If you move a released axis by hand, the firmware still believes the old position — re-home afterwards.
- Printing never needs this command; it is for service work.

---

## `STEPPER_BUZZ` — Wiggle a motor to identify it

### What it does

1. Looks up the motor `STEPPER` (unknown name → the firmware writes `Unknown stepper <name>` to its log and then crashes on the missing motor; the console shows a Go "nil pointer" runtime error instead of a clean message. Nothing moves; this is not a printer shutdown).
2. Remembers whether the motor was enabled, then enables it.
3. Repeats **10 times**: moves the motor **1 mm forward at 4 mm/s** (constant speed, no acceleration), waits 0.05 s, moves **1 mm back**, waits 0.45 s.
4. Restores the previous enable state (disables the motor again if it was off before).

The command ends with the motor back at its starting point, so it is safe even when the printer is not homed. It runs in the movement queue (about 5–6 s in total).

### Syntax

```
STEPPER_BUZZ STEPPER=<name>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Motor to wiggle (**required**). | see the list in the chapter intro | – |

### Examples

```
STEPPER_BUZZ STEPPER=stepper_y1
```
Check which physical motor is the second Y motor (and that it turns the right way: it should first move toward + then back).

### Notes and cautions

- 1 mm is the *motor's* travel in its own units: it is a tiny motion, harmless in practice, but do not run it with the nozzle pressed on the bed.
- Not a G-code that the print needs; use it after changing wiring or a motor.

---

## `FORCE_MOVE` — Move one motor, ignoring all limits

### What it does

1. Looks up the motor `STEPPER`.
2. Enables it (as `STEPPER_BUZZ` does) and moves it by `DISTANCE` at speed `VELOCITY`, with acceleration `ACCEL` (0 = no acceleration, constant speed).
3. Writes the request to the log file.

No end-stop, no travel limit, no homing check: the motor moves exactly the requested amount, even into the frame. The firmware's idea of the toolhead position is **not** updated, so it no longer matches reality (the command help says: *"invalidates kinematics"*). Re-home with `G28` afterwards.

### Syntax

```
FORCE_MOVE STEPPER=<name> DISTANCE=<mm> VELOCITY=<mm/s> [ACCEL=<mm/s²>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Motor to move. | see the list in the chapter intro | – (required) |
| `DISTANCE` | Distance in mm; **negative = opposite direction**. | any number | – (required) |
| `VELOCITY` | Speed in **mm/s** (not mm/min). | > 0 | – (required) |
| `ACCEL` | Acceleration in mm/s². | ≥ 0; 0 = constant speed | 0 |

### Examples

```
FORCE_MOVE STEPPER=stepper_z DISTANCE=5 VELOCITY=5
```
Raise the Z motor by 5 mm at 5 mm/s, with no limit check.

```
FORCE_MOVE STEPPER=stepper_z DISTANCE=-5 VELOCITY=5 ACCEL=100
```
Same distance in the other direction, with a gentle ramp.

### Notes and cautions

- **Enabled by default in this firmware.** The config option `enable_force_move` (section `[force_move]`) defaults to *on* here (it is *off* in standard Klipper), and the stock `printer.cfg` has no such section. The live printer lists `FORCE_MOVE` among its commands, which confirms it.
- Argument errors are reported on the console in this form: `Error on '<your command line>': missing DISTANCE`, `… VELOCITY must have above of 0.000000` (when `VELOCITY` ≤ 0), `… ACCEL must have minimum of 0.000000` (when negative).
- Which direction is "positive" for each motor depends on the wiring/`dir_pin`; try a small distance first.
- After use, send `G28`. **`FORCE_MOVE` does not clear the homed state** (verified: the command never touches the toolhead position or `homed_axes`): the printer keeps believing it is homed at the old position even though the motor has moved.

---

## `SET_KINEMATIC_POSITION` — Declare the toolhead position

### What it does

Sets the position the firmware believes the toolhead is at, **without moving** anything, and marks **X, Y and Z as homed**.

1. Reads `X`, `Y`, `Z`. An omitted value keeps the current kinematic value of that axis.
2. Logs `SET_KINEMATIC_POSITION pos=x,y,z` (log file).
3. Applies the position and flags all three axes as homed.

### Syntax

```
SET_KINEMATIC_POSITION [X=<mm>] [Y=<mm>] [Z=<mm>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `X` | New X position (mm). | number | current position |
| `Y` | New Y position (mm). | number | current position |
| `Z` | New Z position (mm). | number | current position |

### Examples

```
SET_KINEMATIC_POSITION X=0 Y=0 Z=0
```
Pretend the toolhead is at the origin and homed. Typical use: after releasing the motors and placing the head by hand.

### Notes and cautions

- This is a **dangerous** command: the printer will then move relative to a position that may be wrong, ignoring the real end-stops, and may crash into the frame or the bed. Prefer `G28`.
- An omitted axis keeps the toolhead's current value (verified: the defaults are read from the current position). The three axes are marked homed whatever you pass.
- Like `FORCE_MOVE`, available only because `enable_force_move` defaults to on.

---

## `MANUAL_STEPPER` — Drive an extra motor (not on the stock printer)

### What it does

Controls a motor declared in `printer.cfg` as **`[manual_stepper <name>]`** (for example a filament feeder or an extra axis). The **stock config declares none**, so on an unmodified Kobra 3 Max this command has nothing to act on (`STEPPER=<name>` would be unknown).

For a configured one, the command runs these parts in order, each only if its parameter is present:

1. `ENABLE` – switch the motor on/off.
2. `SET_POSITION` – declare its current position (no motion).
3. `STOP_ON_ENDSTOP` – with a non-zero value, perform a homing move to its end-stop (needs `MOVE` and an `endstop_pin` in the config; otherwise error **"No endstop for this manual stepper"**).
4. else `MOVE` – move to the given position at `SPEED`/`ACCEL`; waits for the move to finish when `SYNC=1`.
5. A command with only `SYNC=1` just synchronises the motor with the print queue.

### Syntax

```
MANUAL_STEPPER STEPPER=<name> [ENABLE=<0|1>] [SET_POSITION=<mm>] [MOVE=<mm>]
               [SPEED=<mm/s>] [ACCEL=<mm/s²>] [STOP_ON_ENDSTOP=<n>] [SYNC=<0|1>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Name of the `[manual_stepper]` section. | configured names | – (required) |
| `ENABLE` | Switch motor. | 0 / 1 | unchanged |
| `SET_POSITION` | Declare the current position. | mm | not set |
| `MOVE` | Target position. | mm | no move (with `STOP_ON_ENDSTOP`: 0) |
| `SPEED` | Speed. | mm/s | `velocity` from the section (default 5) |
| `ACCEL` | Acceleration. | mm/s² | `accel` from the section (default 0 = none) |
| `STOP_ON_ENDSTOP` | Homing move to the end-stop. | 0 = normal move; 1 / 2 = stop when triggered (1 errors if never triggered); −1 / −2 = stop when released | 0 |
| `SYNC` | Wait for the move to end before the next command. | 0 / 1 | **1** for a move; 0 when used alone |

### Examples

```
MANUAL_STEPPER STEPPER=feeder ENABLE=1 SET_POSITION=0 MOVE=10 SPEED=5
```
(only if a `[manual_stepper feeder]` section exists) – enable, call this position 0, then move 10 mm.

### Notes and cautions

- Not usable on the unmodified firmware config; documented for completeness. Behaviour is the same as in standard Klipper (parameter list, defaults and error text read from the firmware; nothing run on the printer).
- `STOP_ON_ENDSTOP` values (verified): **positive** = move until the end-stop is *triggered*; **negative** = move until it is *released*; **1** = an error is raised if the end-stop never changes state; **2, −1, −2** = no such error check (note: in standard Klipper −1 also checks; here only `1` does). `MOVE` has no "required" check in a homing move: if omitted it defaults to 0, i.e. the motor homes toward position 0.

---

*End of chapter 15.*


---

# Chapter 16 — Macros (`SET_GCODE_VARIABLE` and the stock macros of `printer.cfg`)

A **macro** is a named G-code command defined in `printer.cfg` as a `[gcode_macro NAME]` section whose `gcode:` lines are run when you type `NAME`. This chapter explains how macros behave in this firmware, then documents the one built-in macro command (`SET_GCODE_VARIABLE`) and the six macros of the stock config. (`M486` is the Exclude-object front end, already explained in chapter 6; only the mapping is repeated here. `CANCEL_PRINT` is also declared as a macro section but belongs to chapter 20.)

| Command | In one line |
|---|---|
| `SET_GCODE_VARIABLE` | Change a variable of a macro (works only on declared variables). |
| `SINGLE_WIPE_NOZZLE` | One back-and-forth wipe of the nozzle on the wiper strip. |
| `WIPE_NOZZLE` | Fan to full, then 3 × `SINGLE_WIPE_NOZZLE`. |
| `MOVE_HEAT_POS` | Lift Z to 15 mm and park the head at the far right (X457). |
| `UNDERLINE` | Print a 40 mm-of-filament purge line along the front edge. |
| `MOVE_END_POS` | Park the head at X0 Y420 (rear left). |
| `M486` | Slicer "object" commands translated to `EXCLUDE_OBJECT*` (see chapter 6). |

### How macros work here

- **Names are not case-sensitive**: the firmware upper-cases the section name, and the stock `WIPE_NOZZLE` itself calls `single_wipe_nozzle` in lower case.
- **Parameters**: when you call a macro as `NAME A=1 B=hello`, the template sees them as `params.A`, `params.B` (names upper-case) and the whole text as `rawparams`. Example: `M486` tests `params.T`, `params.S`…
- **Template language**: the `{% if %}` / `{{ … }}` templates are rendered with a Jinja-like engine (**pongo2**, a Django/Jinja clone), not Python's Jinja2. Variables are written **`{{ name }}`** and logic `{% if … %}` (standard Django/Jinja delimiters; **not** Klipper's single-brace `{name}`). Python expressions and some Jinja2-only features are not available; pongo2 filters such as `integer` and `float` are. The `printer` object (status of the printer objects) and the actions `action_respond_info`, `action_raise_error`, `action_emergency_stop`, `action_call_remote_method` are available inside templates.
- **Layout of the lines**: macro text is split into separate G-code lines using **four consecutive spaces** as the separator (the config indentation). Do not put a run of four or more spaces *inside* a single G-code line in a macro: it would be cut there (verified: after the template is rendered, every run of four spaces is replaced by a line break; extra leading spaces are harmless because blank lines and indentation are ignored).
- **Recursion guard**: a macro that (directly or via others) calls itself stops with the error **`Macro <NAME> called recursively`**. Like the unknown-variable error below, it is returned as an *internal* error, so it puts the printer in the shutdown state.
- **Errors inside a macro** are written to the log (`GCodeMacro Run_gcode_from_command error: …`); they are **not** shown on the console and the macro command itself still reports success. (A failing G-code command *inside* the macro, e.g. a missing parameter, does reach the console.)
- Optional config keys: `description:` (help text) and `rename_existing:` (rename a built-in command so the macro can replace it and still call the original; error `Existing command '…' not found in gcode_macro rename` if the name does not exist).
- **A macro line placed directly in a print file is silently skipped, unless its name starts with `G`/`M`/`T` + a digit (observed on a printer, 2026-11-13; cause found in the firmware code, 2026-11-14).** The print-file reader runs a line only if it starts with `G`+digit, `T`+digit, `M`+digit (the exact line `M84` is refused) or one of 12 prefixes: `SNAPSHOT`, `SET_IDLE_TIMEOUT`, `SET_VELOCITY_LIMIT`, `SET_PRESSURE_ADVANCE`, `SAVE_GCODE_STATE`, `RESTORE_GCODE_STATE`, `TUNING_TOWER`, `SHAPER_CALIBRATE`, `EXCLUDE_OBJECT_DEFINE`, `EXCLUDE_OBJECT_START`, `EXCLUDE_OBJECT_END`, `EXCLUDE_OBJECT`. Any other line (also built-ins such as `BED_MESH_CALIBRATE`, `LEVIQ_*`, `ECHO`) is dropped with no message and no error. Macros called from inside a command that does run work normally (`G9111` runs `MOVE_HEAT_POS`, `G9112` runs `WIPE_NOZZLE`). This is why the stock `M486` macro works from files, and why stock files only use `G9111` for the preparation. A user macro named `M7000` (or any `M`/`G`/`T` + digits) therefore works from a print file; see `ORCA_SLICER_GCODE.md`.
  *(The earlier observation that the job ended with the motors released after a `MOVE_HEAT_POS` line was probably the end of a one-line job; not examined.)*
- No macro here waits for temperature or checks homing: they just send the G-code lines. They assume the printer is **homed** and the **nozzle is clean-able** (hot enough, bed clear).

---

## `SET_GCODE_VARIABLE` — Change a macro variable

### What it does

1. Takes the macro named `MACRO` (multiple macros share this command: the `MACRO=` parameter selects which one).
2. Checks that the macro has a variable named `VARIABLE`. Variables exist only if the macro section declares them with a **`variable_<name>: <value>`** line. If not: error **`Unknown gcode_macro variable '<name>'`** — and, because of how this firmware reports it (returned as an internal error, not as a normal command error), the printer goes into the **shutdown state** ("Internal error on command: SET_GCODE_VARIABLE"): you must switch the printer off and on again before printing.
3. Stores `VALUE` in it. The value is kept as **plain text (a string)** — it is **not** evaluated as a number/list/boolean like in standard Klipper. A macro that compares it with a number must convert it first (e.g. `{% if length|integer > 5 %}`). Variables declared with `variable_<name>:` in the config are text too.
4. The change lasts until the firmware restarts (it is not saved by `SAVE_CONFIG`).

### Syntax

```
SET_GCODE_VARIABLE MACRO=<macro name> VARIABLE=<name> VALUE=<value>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `MACRO` | Macro section to modify (the part after `gcode_macro`). | an existing macro | – (required) |
| `VARIABLE` | Variable name (without the `variable_` prefix). | a declared variable | – (required) |
| `VALUE` | New value, stored as text. | anything | – (required) |

### Examples

With this (user-written) macro in `printer.cfg`:

```
[gcode_macro MY_PURGE]
variable_length: 20
gcode:
    G1 E{{ length }} F300
```

```
SET_GCODE_VARIABLE MACRO=MY_PURGE VARIABLE=length VALUE=30
```
The next `MY_PURGE` purges 30 instead of 20.

### Notes and cautions

- **None of the stock macros declares a variable**, so on the unmodified printer this command always answers *Unknown gcode_macro variable* and **shuts the printer down** (see step 2). **Do not try it on the stock configuration.** It is only useful with your own macros.
- Unknown `MACRO` names are rejected by the command dispatcher (the command is a "multiplexed" one, like `MANUAL_STEPPER`).

---

## `SINGLE_WIPE_NOZZLE` — One wipe stroke

### What it does

Moves the nozzle over the wiper strip at the rear-left of the bed (everything in absolute coordinates, 9600 mm/min = 160 mm/s):

1. `G90` (absolute positioning).
2. Goes to X42 Y430.5 at **Z −0.8** (nozzle pushed below the wiper surface at the edge).
3. Rises to **Z1** while moving to X46, travels along the strip to X98 at Z1.
4. Dips to Z −0.8 at X102, steps over to **Y428**.
5. Returns along the strip at Z1 (X98 → X48), and finishes at X42 Z −0.8.

So the nozzle is dragged across the wiper along two parallel lines (Y430.5 out, Y428 back), pressing in at both ends.

### Syntax

```
SINGLE_WIPE_NOZZLE
```

### Parameters

None.

### Notes and cautions

- Needs the axes **homed**. Z −0.8 is below the printing surface, which is only safe at the wiper position (outside the bed at Y ≈ 428–430); do not change the coordinates without checking.
- It does not heat or cool anything; `WIPE_NOZZLE` / `G9112` handle that. Used by `WIPE_NOZZLE` (and so by the `G9111`/`G9112` start sequence, see chapter 2).

---

## `WIPE_NOZZLE` — Clean the nozzle

### What it does

1. `G90`.
2. `M106 S255`: part-cooling fan to full speed (so the oozing filament hardens and wipes off).
3. Runs `SINGLE_WIPE_NOZZLE` **three times**.

The fan is **not** switched off afterwards: add `M107` (or `M106 S0`) yourself if needed.

### Syntax

```
WIPE_NOZZLE
```

### Parameters

None.

### Example

```
WIPE_NOZZLE
M107
```
Wipe, then turn the fan off.

### Notes and cautions

- Same requirements as `SINGLE_WIPE_NOZZLE`. No temperature check; wiping a cold nozzle just scrapes.

---

## `MOVE_HEAT_POS` — Park for heating

### What it does

1. `G90`.
2. `G1 Z15 F4800`: Z to 15 mm absolute (80 mm/s) — lifts the nozzle off the bed.
3. `M400`: waits until that move is done.
4. `G1 X457 F9600`: moves the head to X457 (far right, the same X as the pause position in `[pause_resume]`).

### Syntax

```
MOVE_HEAT_POS
```

### Parameters

None.

### Notes and cautions

- Z is absolute: if Z is **not homed** the move is unreliable. X457 is beyond the 420 mm bed but inside the X limit (478).
- The Y position is not changed.

---

## `UNDERLINE` — Purge line

### What it does

1. `G90`; moves the nozzle to **Z0.28**.
2. `G92 E0`: zeroes the extruder position.
3. `G1 Y1 F2400`: goes to Y1 (front edge) at 40 mm/s. **X is not changed**: the line starts at the current X.
4. `G1 X180 E40 F500`: draws a line to **X180** while extruding **40 mm of filament**, slowly (500 mm/min ≈ 8.3 mm/s).
5. `M400`, `G92 E0`, `G1 E-0.5 F300`: waits, zeroes E, **retracts 0.5 mm** (300 mm/min = 5 mm/s).
6. `G1 Z5`, `M400`: lifts to Z5 and waits.

### Syntax

```
UNDERLINE
```

### Parameters

None.

### Notes and cautions

- The result is a purge line along the front edge (Y1), from the current X to X180. In the `G9111` start sequence `UNDERLINE` runs right after `MOVE_HEAT_POS` (verified call order), so it **starts at X457** and runs to X180: 277 mm long, of which the part beyond the 420 mm bed edge (X420–457) lies off the bed. The 40 mm of filament is spread along the whole path (≈ 0.14 mm of filament per mm). Called by hand it starts wherever the head is.
- Requires a hot nozzle and homed axes. As noted in chapter 2, `G9111` runs it only when the ACE filament hub is active (flag inferred from the firmware; not tested with an ACE).

---

## `MOVE_END_POS` — Park at the rear

### What it does

`G90`, `G1 X0 Y420 F9600` (160 mm/s) and `M400`. Parks the head at the **rear-left** corner (Y420 is the back of the bed), waiting for the move to finish. Z is not changed.

### Syntax

```
MOVE_END_POS
```

### Parameters

None.

### Notes and cautions

- Because Z is untouched, lift Z first (e.g. `MOVE_HEAT_POS` or `G1 Z…`) if the nozzle is near the bed.

---

## `M486` — Object commands (front end of Exclude object)

Declared as a macro so slicers that emit `M486` work. It translates its parameters into `EXCLUDE_OBJECT*` commands; see chapter 6 for what each does:

| `M486` form | Becomes |
|---|---|
| `M486 T<n>` | `EXCLUDE_OBJECT RESET=1`, then `EXCLUDE_OBJECT_SET_OBJ T=<n>` |
| `M486 C` | `EXCLUDE_OBJECT CURRENT=1` |
| `M486 P<name>` | `EXCLUDE_OBJECT NAME=<name>` (cancel that object) |
| `M486 S<name>` | `EXCLUDE_OBJECT_START NAME=<name>`; `M486 S-1` → `EXCLUDE_OBJECT_END_NO_OBJ` |
| `M486 U<name>` | `EXCLUDE_OBJECT RESET=1 NAME=<name>` (un-cancel) |

Notes: each parameter is tested for being present/non-empty, and the values are text, so a value of `0` (the string "0") counts as present: `M486 S0` and `M486 P0` work. `M486 S-1` is matched as the text "-1" (pongo2 rule: non-empty text is true; from the library's semantics, not tested on the printer).

---

*End of chapter 16.*

---

# Chapter 17 — Misc (`SAVE_CONFIG`, `SET_IDLE_TIMEOUT`, `SET_DUAL_CARRIAGE`, `M1011`)

Four unrelated commands. Two of them (`SET_DUAL_CARRIAGE`, `M1011`) only exist when an optional section is present in the configuration, which the stock Kobra 3 Max configuration does not have, so on a stock printer you will normally not be able to use them. (`BED_MESH_PROFILE`, listed under "Misc" in the command table, is in chapter 3.)

| Command | In one line |
|---|---|
| `SAVE_CONFIG` | Writes the staged changes (bed mesh, Z offset, input shaper, PID…) to the configuration file, with a backup. Does not restart the firmware. |
| `SET_IDLE_TIMEOUT` | Changes how many seconds of inactivity pass before the printer turns heaters, fan and motors off. Temporary. |
| `SET_DUAL_CARRIAGE` | Chooses the active carriage of a dual-carriage machine. Not available on the Kobra 3 Max. |
| `M1011` | Switches an LED output on or off. Not available on the stock configuration. |

---

## `SAVE_CONFIG` — Save staged settings to the configuration file

### What it does

Many commands only **stage** a change: `BED_MESH_PROFILE SAVE=`, `BED_MESH_CALIBRATE`, `Z_OFFSET_APPLY_PROBE`, `SHAPER_CALIBRATE`, `PID_CALIBRATE`, `LEVIQ_AUTO_ZOFFSET`… They keep the new values in memory, together with a note "to be saved". `SAVE_CONFIG` is the command that makes them permanent:

1. Reads the current configuration file from disk.
2. Adds the staged values to the machine-written block at the end of the file (the block that starts with `#*# <--- SAVE_CONFIG --->` and where every line begins with `#*#`). Values already in that block are replaced; the rest of the file is left alone.
3. Checks that the result can still be read as a valid configuration.
4. Makes a **timestamped backup** of the old file next to it (name = file name + `-YYYYMMDD_HHMMSS`), writes the new content to a temporary `…_autosave.cfg` file, then renames it over the real file.
5. Reloads the configuration and tells the other parts of the firmware that the configuration was saved.

On a printer with a separate "mutable" configuration file (Rinkhals-style, `printer_mutable.cfg`) the values go to that file and `printer.cfg` is left alone. The firmware also keeps a copy of some default LeviQ data in the same way (internal; no visible effect).

### Syntax

```
SAVE_CONFIG
```

### Parameters

None.

### Examples

Keep a freshly measured bed mesh:

```
BED_MESH_CALIBRATE
SAVE_CONFIG
```

### Notes and cautions

- **No restart.** The help text says "Overwrite config file and restart" (inherited from Klipper), but in this firmware the command only writes the file and reloads; it does not restart. This is why macros such as `G9111` can call it in the middle of a print start.
- **Status displays can be misleading.** On a Rinkhals-style printer, Moonraker's `configfile` object may keep showing the old value and the "pending" list after a successful save (observed, see chapter 3). Read the file, or `RESTART`, to be sure.
- **Only staged items are written.** A value changed live but not staged (for example `SET_GCODE_OFFSET`, `LEVIQ_SET_ZERO`, `SET_INPUT_SHAPER`) is **not** saved by `SAVE_CONFIG`.
- Every run leaves a backup file next to the configuration file. The firmware also cleans up old backups of the main file (to verify: how many it keeps).
- If the existing file cannot be parsed the command fails with `Unable to parse existing config on SAVE_CONFIG`. In this firmware that kind of error is reported as an internal error and puts the printer in shutdown (needs a power off/on) — a rare case (to verify).
- There is no check for a running print. Avoid it during a print unless you know why: it writes to flash and reloads the configuration.

---

## `SET_IDLE_TIMEOUT` — Change the inactivity timeout

### What it does

The firmware watches the toolhead. When nothing has been printed or moved for the **idle timeout** (default **600 s = 10 minutes**; there is no `[idle_timeout]` section in the stock configuration), it runs a shut-down routine:

```
TURN_OFF_HEATERS
M107
M1011 S0
M84
```

That is: heaters off, part fan off, LED output off (only if one is configured — otherwise this line is just an unknown command, to verify), motors off (the axes become un-homed, see chapter 15). While a print job is running the routine is skipped (a message goes to the log only). It is also postponed while another G-code script is running.

`SET_IDLE_TIMEOUT` changes that delay:

1. Reads `TIMEOUT` (must be greater than 0) and stores it.
2. Prints `idle_timeout: Timeout set to <value> s` on the console.
3. If the printer is currently in its "Ready" state (not printing, not yet idle), restarts the countdown with the new value counted from now.

### Syntax

```
SET_IDLE_TIMEOUT [TIMEOUT=<seconds>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `TIMEOUT` | Seconds of inactivity before the shut-down routine runs. | Number greater than 0 | The current timeout (so the command alone just reports it) |

### Examples

Keep heaters on for 30 minutes of inactivity:

```
SET_IDLE_TIMEOUT TIMEOUT=1800
```

Report the current value:

```
SET_IDLE_TIMEOUT
```

### Notes and cautions

- **Temporary:** the value is lost at restart. To change it permanently add an `[idle_timeout]` section with `timeout:` to the configuration file (the firmware reads a `timeout` option, default 600; not tested).
- Very large values keep a hot nozzle and bed on unattended for a long time. Leave the default unless you have a reason.
- In the firmware the status object `idle_timeout` would report `state` (Idle / Ready / Printing) and `printing_time` (seconds since printing started, 0 otherwise). **On the live printer the object exists but Moonraker returns it empty (`{}`)**, so you cannot read the state from Mainsail (the reason is unknown, to verify).
- **Tested on the printer (idle, cold, nothing printing):** `SET_IDLE_TIMEOUT`, `SET_IDLE_TIMEOUT TIMEOUT=600` and `SET_IDLE_TIMEOUT TIMEOUT=0` were each accepted (`ok`) but produced **no console message at all** — no `idle_timeout: Timeout set to … s`, and no error for the invalid value 0. A made-up command (`FOO_BAR_TEST`) behaves identically (silent), while a real command with a missing argument (`SET_STEPPER_ENABLE`) does print its error. Together with its absence from the `/printer/gcode/help` list and the empty `idle_timeout` status, this strongly suggests that **on this printer the command is not active** (the firmware in use behaves as if it were unknown; the code is present in the binary, but it is not reachable here). Treat the description above as what the binary contains, not as proven behaviour; whether the 600 s shut-down exists at all on this printer is still unverified.
- Do not confuse it with the `idle_timeout: 30` line in the stock `[controller_fan]` section: that one is the fan's own delay (seconds the controller fan keeps running after the motors stop) and has nothing to do with this command.

---

## `SET_DUAL_CARRIAGE` — Select the active carriage

### What it does

Only exists when the configuration has a `[dual_carriage]` section (a second carriage on the X or Y axis, as on IDEX printers). It makes carriage 0 (normal) or carriage 1 (the dual) the active one: the toolhead position is reset for the chosen carriage and the axis limits become those of that carriage.

The Kobra 3 Max has no `[dual_carriage]` section (its two Y motors, `stepper_y` and `stepper_y1`, move the bed together), so **the command is not registered and gives "Unknown command"** — the live printer's command list does not contain it.

### Syntax

```
SET_DUAL_CARRIAGE CARRIAGE=<0|1>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `CARRIAGE` | Which carriage becomes active. | 0 or 1 | None (required) |

### Notes and cautions

- Do not add a `[dual_carriage]` section to this machine: it has no second carriage.
- The carriage switching itself was only traced in outline (to verify).

---

## `M1011` — LED on / off

### What it does

Drives an LED output made of two pins. It is registered only if the configuration has the matching LED section. The stock configuration has none and the live printer shows no LED object, so on a stock Kobra 3 Max the command is probably **unknown** (to verify; the exact section name could not be extracted, probably `[led …]`).

When available:

1. `S` must be 0 or 1, otherwise an error is logged (`led status invalid`) and nothing changes.
2. Sets the first pin and then the second pin to the requested level (each change scheduled 0.1 s after the current time).
3. Remembers the state and tells the printer's "LED detect" feature (internal).

It is also called by the idle-timeout routine (`M1011 S0`, see above), and a web request `led/set_led` exists for the touchscreen.

### Syntax

```
M1011 [S<0|1>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `S` | LED state. | 0 = off, 1 = on | 0 |

### Examples

```
M1011 S1
M1011 S0
```

### Notes and cautions

- `M1011` without `S` switches the LED **off**.
- Whether the printer's work light is controlled this way could not be established.

---

*End of chapter 17.*


---

# Chapter 18 — Motion state: moves and modes (`G0`, `G1`, `G2`, `G3`, `G17`, `G18`, `G19`, `G20`, `G21`, `G90`, `G91`, `M82`, `M83`)

These are the everyday commands of every G-code file: **move the toolhead** (`G0`/`G1`, plus arcs `G2`/`G3`) and **choose how the numbers are read** (absolute or relative, which plane an arc lies in, units). Position, offsets, saved state and speed/flow factors (`G92`, `M114`, `M220`, `M221`, `SAVE_GCODE_STATE`…) are in chapter 19.

| Command | In one line |
|---|---|
| `G0` / `G1` | Move to a position (and/or extrude), at a given speed. Both are **the same command**. |
| `G2` / `G3` | Move along an arc, clockwise / counter-clockwise (cut into short straight segments). |
| `G17` / `G18` / `G19` | Choose the plane of the arcs: X-Y / X-Z / Y-Z. |
| `G20` / `G21` | Inches / millimetres. **No effect** on this printer (see below). |
| `G90` / `G91` | Absolute / relative positioning of the axes. |
| `M82` / `M83` | Absolute / relative positioning of the extruder (`E`). |

> **Units:** the printer works in **millimetres** only. `F` is a feed rate in **mm per minute** (the unit of the G-code file); the speeds written in `printer.cfg` are in mm/s.

---

## `G0` / `G1` — Linear move

### What it does

`G0` and `G1` are **identical**: the firmware makes no difference between a "rapid" and a "working" move.

1. For each of `X`, `Y`, `Z` that is present, computes the new target:
   - in **absolute** mode (`G90`, the usual mode): the target is the number you gave, plus the current coordinate offset (see `G92` and `SET_GCODE_OFFSET`, chapter 19);
   - in **relative** mode (`G91`): the number is **added** to the previous target.
2. If `E` is present, computes the new extruder target, multiplied by the extrusion factor (`M221`, chapter 19):
   - **absolute** if *both* `G90` and `M82` are active (the usual case in a print);
   - **relative** (added to the previous position) if `G91` **or** `M83` is active.
3. If `F` is present, sets the **speed** (see below). The speed is remembered: following moves without `F` reuse it.
4. Sends the move to the planner. The command returns immediately; the motion runs when the queued moves before it are finished. (Use `M400` to wait for the end of the motion.)

An axis that is **not** in the command keeps its previous target. `G1` alone, with no parameter, does not move anything.

### Syntax

```
G1 [X<mm>] [Y<mm>] [Z<mm>] [E<mm>] [F<mm/min>]
G0 [X<mm>] [Y<mm>] [Z<mm>] [E<mm>] [F<mm/min>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `X` | Target X (absolute) or distance (relative). | mm; the stock X range is about −28 … 478 | axis not moved |
| `Y` | Target Y (absolute) or distance (relative). | mm; stock range about −5.5 … 440 | axis not moved |
| `Z` | Target Z (absolute) or distance (relative). | mm; stock range about −4 … 502 | axis not moved |
| `E` | Filament to push (positive) or pull back (negative), as an absolute position (`M82`) or a distance (`M83`). | mm of filament | no extrusion |
| `F` | Feed rate for this and the following moves. | mm/min, e.g. `F6000` = 100 mm/s | previous speed |

### How the speed is decided

1. `F` is multiplied by the speed factor (`M220`, chapter 19; 100 % by default) and converted to mm/s.
2. **Halving rule (to verify on a real print):** if the printer is **printing** *and* the file declared a G-code *resolution* **smaller than 0.012** (a line `; resolution = …` or `; gcode_resolution = …` in the file; `0` is read as 0.0125), the speed is **halved**. Files from Anycubic Slicer Next declare `; resolution = 0.012`, which is *not* smaller, so nothing changes for them. For a file that declares nothing, the firmware's default is 0.012 as well, but the reading of the first 30 lines of the file may leave 0 there (see the notes).
3. If the result is **0.001 mm/s or less** (e.g. `F0`, a negative `F`, or text that is not a number), the speed becomes **40 mm/s** (= `F2400`).

The check is done only when the command contains `F`. The resulting speed is then still limited by the printer's own limits (`max_velocity` 600 mm/s, acceleration 10 000 mm/s² in the stock configuration — see `SET_VELOCITY_LIMIT`, chapter 28).

### Examples

```
G90
G1 X100 Y200 F6000
```
Absolute mode; move to X100 Y200 at 100 mm/s.

```
G91
G1 Z5 F600
G90
```
Relative mode; lift 5 mm at 10 mm/s; back to absolute.

```
G1 X150 E2.5 F1800
```
Move to X150 pushing 2.5 mm of filament, at 30 mm/s (in a print with `M82` the `E` is the new absolute extruder position, not a length to add).

```
M83
G1 E-2 F2400
```
Relative extrusion: retract 2 mm of filament at 40 mm/s.

### Notes and cautions

- **Wrong or missing numbers count as 0.** A value that is not a valid number (e.g. `G1 X abc`) is written to the log file and then **read as 0**; the console shows no error. With `F`, 0 means the 40 mm/s default. Do not rely on this.
- **Limits (standard behaviour, not specific to Anycubic):** moving an axis that is not homed gives **`Must home axis first`**; moving outside the machine area gives **`Move out of range`**. In the stock configuration an extrusion is refused below **170 °C** (`Extrude below minimum temp`), an extrusion-only move longer than **100 mm** gives `Extrude only move too long`, and a move whose extrusion cross-section exceeds **50 mm²** gives `Move exceeds maximum extrusion`. These errors appear on the console and the command is not executed.
- **Long retraction warning:** if a single move pulls the filament back by **more than 10 mm** the firmware writes `retraction maybe too long, retract distance=<n>` to its log (warning only; the move is done).
- Relative extrusion is also selected by `G91` (not only `M83`): after `G91` an `E` value is always a distance, even if `M82` was sent before.
- The printer reads the first 30 lines of a G-code file to recognise the slicer. If none of them carries a `; generated by …` comment, the resolution may be read as 0 for the halving rule, until a `; resolution =` line is found later in the file. **Whether this really slows such files down could not be confirmed by reading the firmware alone (to verify)**: if a file from another slicer prints at about half the programmed speed, add `; resolution = 0.012` to its header.
- Unlike stock Klipper, an `F` of 0 or less is **not** an error here: it silently selects the 40 mm/s default.

---

## `G2` / `G3` — Arc move

### What it does

`G2` draws a **clockwise** arc, `G3` a **counter-clockwise** arc, from the current position to the target. The firmware does not move along a real curve: it cuts the arc into short straight `G1` moves.

1. Refuses to run in **relative mode**: error **`G2/G3 does not support relative move mode`** (send `G90` first).
2. Reads the target `X`, `Y`, `Z` (an axis that is missing stays at its current value) and the centre offsets (below). If the `R` parameter is present: error **`G2/G3 does not support R moves`**. If the centre offsets are all 0 (or missing): error **`G2/G3 requires IJ, IK or JK parameters`**.
3. Computes the arc around the centre `current position + offsets`. If the target is the same point as the start (and the angle is 0), a **full circle** is drawn.
4. If the third axis of the plane (the "helical" axis) changes, the arc becomes a spiral: that axis moves a constant fraction in each segment.
5. Cuts the arc into segments of about **1 mm** (`resolution` in `[gcode_arcs]` of `printer.cfg`; at least one segment). Each segment is sent as a `G1`, so everything described for `G1` (speed, limits, warnings) applies to each segment. The **last segment ends exactly on the requested target**.
6. If `E` is given, it is the extruder position **at the end of the arc** (`M82`) or the length for the **whole arc** (`M83`): the firmware spreads it equally over the segments. If `F` is given, it is passed to every segment (and so stays active afterwards).

### Syntax

```
G2 [X<mm>] [Y<mm>] [Z<mm>] I<mm> J<mm> [E<mm>] [F<mm/min>]      (plane G17, default)
G3 [X<mm>] [Y<mm>] [Z<mm>] I<mm> J<mm> [E<mm>] [F<mm/min>]
G2 [X<mm>] [Y<mm>] [Z<mm>] I<mm> K<mm> ...                      (plane G18)
G2 [X<mm>] [Y<mm>] [Z<mm>] J<mm> K<mm> ...                      (plane G19)
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `X`, `Y`, `Z` | Arc end point (absolute coordinates). | mm | current position of that axis |
| `I`, `J`, `K` | Offset **from the start point to the arc centre** along X, Y, Z. The pair needed depends on the plane: `I`,`J` for G17; `I`,`K` for G18; `J`,`K` for G19. | mm, positive or negative | 0 |
| `E` | Extrusion, see step 6. | mm of filament | none |
| `F` | Feed rate, as for `G1`. | mm/min | previous speed |
| `R` | Radius form — **not supported**: gives an error. | – | – |

### Examples

```
G90
G1 X100 Y100 F6000
G2 X120 Y100 I10 J0 F1200
```
From (100,100), a clockwise half circle of 10 mm radius ending at (120,100); centre at (110,100).

```
G3 X100 Y100 I10 J0 E1.5
```
Counter-clockwise arc back to the start, pushing filament (E is absolute with `M82`).

```
G2 X50 Y50 I10 J0
```
End point = start point: a full circle of 10 mm radius.

### Notes and cautions

- The radius is taken from the centre offsets, so the start point is always exactly on the circle; the **end point is not checked**: if it is not on the same circle, the arc just ends on it with a spiral-like final segment.
- Small arcs (shorter than the 1 mm resolution) become a single straight line.
- The arc commands are **not used by the Anycubic slicer**; they are mostly useful for hand-written G-code.
- **Planes G18 and G19 are faulty (confirmed in the firmware code; not run on a printer).** The centre and the angles are computed correctly for the chosen plane, but each intermediate point is then sent to `X`, `Y`, `Z` in that fixed order, whatever the plane. In plane G17 this is correct. In G18 (X-Z) the value meant for Z is sent to Y and the value meant for Y is sent to Z; in G19 (Y-Z) the three values are likewise sent to the wrong axes. Only the **last** point (the requested end point) is right, so the toolhead follows a wrong path to it. Do not use `G18`/`G19` arcs on this printer; do the arc in G17 or use straight `G1` moves.
- The three error messages above appear on the console only; they do **not** put the printer in shutdown.

---

## `G17` / `G18` / `G19` — Arc plane

### What it does

Chooses the plane in which `G2`/`G3` arcs are drawn, until changed again:

| Command | Plane | Centre offsets needed | Third (helical) axis |
|---|---|---|---|
| `G17` | X-Y (default) | `I`, `J` | Z |
| `G18` | X-Z | `I`, `K` | Y |
| `G19` | Y-Z | `J`, `K` | X |

No motion, no parameters, no message. The setting is kept in memory only; at power-up or after a restart the plane is **G17**. These commands have no other effect and the printer never uses them itself.

### Syntax

```
G17
G18
G19
```

### Notes and cautions

- See the warning under `G2`/`G3` about the G18/G19 planes.
- The plane is not part of the state saved by `SAVE_GCODE_STATE` (to verify; chapter 19).

---

## `G20` / `G21` — Inches / millimetres

### What they do

**Nothing.** The printer only works in millimetres.

- `G20` (inches) is **not supported**: it writes `Machine does not support G20 (inches) command` to the log file. It does **not** change the units and it does **not** stop the print or show an error on the console (the console shows just `ok`). The following numbers are still read as millimetres.
- `G21` (millimetres) also writes `Machine does not support G21 command` to the log file, and does nothing, which is correct since millimetres are already used. Many slicers put `G21` at the start of the file; this is harmless.

### Syntax

```
G20
G21
```

### Notes and cautions

- A file written in inches would print about 25 times too small without any warning: check the slicer's unit setting if a model comes out wrong.

---

## `G90` / `G91` — Absolute / relative positioning

### What they do

- `G90` — **absolute** mode: the `X`, `Y`, `Z` of the moves are positions on the bed (default; the slicer sends it at the start of the file).
- `G91` — **relative** mode: the numbers are distances from the previous position (useful for small jogs, as in `HOME_Y` of the stock macros).

In `G91` the extruder is relative as well, whatever `M82`/`M83` say. `G90` alone does not change the extruder mode: if you used `M83` (relative extrusion) it stays relative after `G90` until `M82` is sent.

The mode is the same for all moves of `G0`/`G1`; `G2`/`G3` only work in `G90`. Homing (`G28`) and the macros that need a certain mode send `G90`/`G91` themselves.

### Syntax

```
G90
G91
```

### Examples

```
G91
G1 Z10 F600
G90
```
Lift 10 mm, then return to absolute mode.

### Notes and cautions

- Forgetting `G90` after `G91` is the classic cause of a "Move out of range" error or of a crash: send `G90` at the end of every relative sequence (the stock macros do).
- The mode is saved/restored by `SAVE_GCODE_STATE` / `RESTORE_GCODE_STATE` (chapter 19).

---

## `M82` / `M83` — Absolute / relative extrusion

### What they do

- `M82` — the `E` values are **absolute positions** of the extruder (the slicer sends `M82` and counts the filament used since the last `G92 E0`). This is the mode Anycubic Slicer Next uses.
- `M83` — the `E` values are **lengths to push** in each move (positive = extrude, negative = retract).

These commands set only the extruder part. They are combined with `G90`/`G91` as in step 2 of `G1`: absolute extrusion needs **both** `G90` and `M82`.

### Syntax

```
M82
M83
```

### Examples

```
M83
G1 E5 F300
G1 E-5 F1800
```
Push 5 mm of filament slowly, then pull it back 5 mm, whatever the previous E position.

### Notes and cautions

- Switching between `M82` and `M83` does not move anything and does not reset the extruder position (use `G92 E0`, chapter 19).
- In `M82` mode an `E` lower than the current position is a **retraction**; a very large `E` jump (for instance after you forgot `G92 E0` when changing the mode) is a very long extrusion: the "extrude only move too long" limit (100 mm) will stop it.

---

*End of chapter 18.*


---

# Chapter 19 — Position, offsets, saved state, speed and flow (`G92`, `M114`, `GET_POSITION`, `SET_GCODE_OFFSET`, `SAVE_GCODE_STATE`, `RESTORE_GCODE_STATE`, `G60`, `G61`, `M220`, `M221`, `M222`)

Chapter 18 explained how a move is read. These commands deal with **what the numbers mean**: shifting the coordinate system (`G92`, `SET_GCODE_OFFSET`), reading the position back (`M114`, `GET_POSITION`), saving and restoring the whole "G-code state" (`SAVE_GCODE_STATE` and friends), and scaling speed and extrusion (`M220`, `M221`).

Vocabulary used below:

- **G-code position** — the position as the G-code sees it: the last *target* sent, minus the offsets. It is not the measured position of the motors.
- **Offset** — a number added to every coordinate of an axis. Two kinds exist: the ones made by `G92` and the "homing offset" made by `SET_GCODE_OFFSET` (for instance the Z offset).

| Command | In one line |
|---|---|
| `G92` | Declare that the current position is a given value (shift the coordinate system, no motion). |
| `SET_GCODE_OFFSET` | Set a permanent offset (for example Z) that is added to all moves; optionally move by it. |
| `M114` | Print the current G-code position. |
| `GET_POSITION` | Print a detailed position report (service/diagnostic). |
| `SAVE_GCODE_STATE` / `G60` | Save the G-code state (positions, modes, speed, factors) under a name. |
| `RESTORE_GCODE_STATE` / `G61` | Restore a saved state; optionally move back to the saved position. |
| `M220` | Speed factor in percent. |
| `M221` | Extrusion (flow) factor in percent. |
| `M222` | Reset speed, factors and modes to defaults. |

---

## `G92` — Set position (shift the coordinate system)

### What it does

`G92` does **not move anything**. It tells the firmware: "the toolhead (or filament) that you believe is at position *P* is now at the value I give". Internally the difference is stored as an offset, so that all following moves are shifted.

1. For each of `X`, `Y`, `Z`, `E` that is present, the offset is set so that the **current position reads as the given value**. For `E` the value is first multiplied by the extrusion factor (`M221`).
2. If **none** of the four is present, all four axes are set to **0** (the current position becomes the origin for X, Y, Z and E).

### Syntax

```
G92 [X<mm>] [Y<mm>] [Z<mm>] [E<mm>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `X`, `Y`, `Z` | The value the current position of that axis should read as. | mm | axis unchanged |
| `E` | The value the extruder position should read as. | mm of filament | unchanged |

(With no parameter at all: X, Y, Z and E are all set to 0.)

### Examples

```
G92 E0
```
Reset the extruder position to 0 (every layer change in a slicer file that uses absolute `E`).

```
G92 X0 Y0
```
The current XY position becomes the origin of the G-code coordinates. The toolhead does not move; later `G1 X10` moves 10 mm further from here.

```
G92
```
Everything (X, Y, Z, E) reads as 0 here.

### Notes and cautions

- `G92` does **not** home the printer or change the homed state; it only shifts the numbers. The limits of the machine (`Move out of range`) are still checked in the machine's own coordinates.
- **Dangerous on X, Y, Z in the middle of a print:** all following moves are shifted. Use it only on `E` unless you know exactly why.
- **Homing cancels the shift** of the homed axes: after `G28` (or `G28 X`, …) the `G92` shift of each axis that was homed is gone, and only the `SET_GCODE_OFFSET` offset remains. Send `G92` again after homing if you need it.
- `M114` shows the shifted numbers; `GET_POSITION` shows the offsets ("gcode base").

---

## `SET_GCODE_OFFSET` — Permanent offset (e.g. Z offset)

### What it does

Sets a **homing offset** per axis: a number added to all coordinates sent by the G-code. The main use is the baby-step *Z offset* (nozzle height above the bed). The offset is a *setting of the current session*: it is not written to `printer.cfg`.

1. For each axis `X`, `Y`, `Z`, `E`:
   - if the axis parameter (e.g. `Z`) is present, the offset **becomes** that value;
   - otherwise, if the `…_ADJUST` parameter (e.g. `Z_ADJUST`) is present, that value is **added to the current offset**;
   - otherwise the axis is not changed.
2. If `MOVE=1`, the toolhead then moves by the change of offset (so that it keeps its position relative to the part), at `MOVE_SPEED`.

### Syntax

```
SET_GCODE_OFFSET [X=<mm>] [Y=<mm>] [Z=<mm>] [E=<mm>]
                 [X_ADJUST=<mm>] [Y_ADJUST=<mm>] [Z_ADJUST=<mm>] [E_ADJUST=<mm>]
                 [MOVE=<0|1>] [MOVE_SPEED=<mm/s>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `X`, `Y`, `Z`, `E` | Set the offset of that axis to this value. | mm | unchanged |
| `X_ADJUST`, `Y_ADJUST`, `Z_ADJUST`, `E_ADJUST` | Add this to the offset of that axis (ignored if the plain parameter is also given). | mm | – |
| `MOVE` | Also move the toolhead by the change. | 0 = no, 1 (or any non-zero) = yes | 0 |
| `MOVE_SPEED` | Speed of that move. **Unit: mm/s** (not mm/min like `F`). | mm/s | the current speed |

### Examples

```
SET_GCODE_OFFSET Z=0.1
```
Raise the nozzle by 0.1 mm on all following moves (not applied to the position right now).

```
SET_GCODE_OFFSET Z_ADJUST=-0.02 MOVE=1
```
Lower by 0.02 mm and also move the nozzle down now (a "baby step" during the first layer).

```
SET_GCODE_OFFSET Z=0
```
Remove the offset.

### Notes and cautions

- The offset is **lost at a restart** of the printer software (after a `RESTART` the value read back was 0).
- To keep a Z offset permanently use the probe offset commands (`Z_OFFSET_APPLY_PROBE` followed by `SAVE_CONFIG`, chapter 24 and chapter 17).
- `LEVIQ_SET_ZERO` (chapter 14) sets this offset back to 0.
- Without `MOVE=1` the toolhead only gets the new offset at its **next** move.

---

## `M114` — Report the position

### What it does

Prints one line with the **G-code position** of X, Y, Z and E, for example:

```
X:100.000 Y:50.000 Z:0.300 E:12.345
```

The numbers are the last **target** sent to the planner (offsets removed; `E` divided by the extrusion factor). It is **not** a measurement and the command does **not wait** for the moves in the queue: while the printer is still moving, the values are where it will be after the moves already queued.

### Syntax

```
M114
```

### Parameters

None.

### Notes and cautions

- To get the position *after* the motion has finished, send `M400` first (chapter 28).
- The reply is plain text on the console, with three decimals.
- For the machine-level view (motor positions) use `GET_POSITION`.

---

## `GET_POSITION` — Detailed position report

### What it does

Prints a multi-line report with the positions seen by each layer of the firmware: the motor step counters, the commanded position of each motor, the position computed from the motors, the toolhead position, the G-code position, the G-code "base" offsets and the homing offsets. It is a diagnostic tool, mainly useful to compare layers (e.g. after a lost step or a wrong offset).

Format of the report:

```
mcu: stepper_x:<steps> stepper_y:<steps> ...
stepper: stepper_x:<mm> stepper_y:<mm> ...
kinematic: X:[<x> <y> <z>] Y:[<x> <y> <z>] Z:[<x> <y> <z>]
toolhead: X:<..>Y:<..>Z:<..>E:<..>
gcode: X:<..>Y:<..>Z:<..>E:<..>
gcode base: X:<..>Y:<..>Z:<..>E:<..>
gcode homing: X:<..>Y:<..>Z:<..>
```

Details (verified in the code): the `mcu` and `stepper` lines list the motors (`%d` steps, then `%.6f` mm); the `kinematic` line prints the **same** position vector (X, Y, Z computed from the motors) once under each of the labels X, Y and Z, **in random order** from one call to the next; the four lines `toolhead`, `gcode`, `gcode base` and `gcode homing` have no space between the values (`X:100.000000Y:50.000000Z:0.300000E:12.000000`) and use 6 decimals. `gcode` is the planned G-code target before the offsets are removed, so it is **not** equal to `M114` when offsets are active (`M114` = `gcode` − `gcode base`).

### Syntax

```
GET_POSITION
```

### Parameters

None.

### Notes and cautions

- The report is printed to the console and the log; the layout is untidy (values glued together, `kinematic` repeated three times in random order, see above). Read it as a debug dump, not as a stable format.
- If the printer is not ready: error **`Printer not ready`**.
- It does not move anything and does not wait for the moves in the queue.

---

## `SAVE_GCODE_STATE` / `G60` — Save the state

### What it does

Stores a snapshot of the G-code state under a name (kept in memory only):

- absolute/relative mode of the axes (`G90`/`G91`) and of the extruder (`M82`/`M83`),
- the current G-code target positions (X, Y, Z, E), the `G92` offsets and the `SET_GCODE_OFFSET` offsets,
- the current speed, the speed factor (`M220`), the extrusion factor (`M221`), and the speed mode value (see `M222`).

`G60` is **the same command** as `SAVE_GCODE_STATE` (same behaviour, same storage, same default name): a state saved with `G60` can be restored with `RESTORE_GCODE_STATE` and the other way round.

A snapshot with the same name is **overwritten**. The temperature, the fans and the **arc plane** (`G17`/`G18`/`G19`) are **not** part of the state.

### Syntax

```
SAVE_GCODE_STATE [NAME=<name>]
G60 [NAME=<name>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `NAME` | Name of the snapshot. | any text | `default` |

### Examples

```
SAVE_GCODE_STATE NAME=before_wipe
G91
G1 Z5 F600
G1 X20 F6000
RESTORE_GCODE_STATE NAME=before_wipe MOVE=1
```
Save the state, make some relative moves, then return to the saved position and mode.

### Notes and cautions

- This is used by the pause (`PAUSE` saves the state as `PAUSE_STATE`, chapter 21) and by many macros.
- Snapshots are lost when the printer software restarts.

---

## `RESTORE_GCODE_STATE` / `G61` — Restore the state

### What it does

Takes the snapshot saved under `NAME` and puts the modes, offsets, speed and factors back. Unless `MOVE=1` is given, **nothing moves**.

1. Unknown name: error **`Unknown g-code state: <name>`** (nothing changes).
2. Restores `G90`/`G91`, `M82`/`M83`, the offsets, the speed, the speed factor, the extrusion factor and the speed mode.
3. **Extruder position** (this is where the two commands differ):
   - `RESTORE_GCODE_STATE`, when **both** `G90` and `M82` are active after restoring (the usual print mode): the extruder's G-code position is set back to the **saved** value and the offsets of the snapshot are put back as they were, **without moving the extruder**. The next absolute `E` is therefore measured exactly as before the save, so every extruder move that was made **with G-code `E` values** between the save and the restore is **cancelled out by the first extruding move afterwards**: filament that was pushed (for example a purge with `G1 E20` in a macro) is pulled back, filament that was retracted is pushed back. (Verified in the code; not tested on a printer.)
   - The pause is a special case: the firmware's own pause moves the extruder with direct motor moves that the G-code position does not see, so the G-code `E` position is unchanged at the restore, and the first extruding move after `RESUME` goes from the real (retracted) extruder position to the file's `E` value, whichever restore variant is used.
   - `RESTORE_GCODE_STATE`, in relative modes (`G91` or `M83` active), and `G61` in all modes: the extruder offset is shifted so that the extruder simply continues from where it is now; what happened to the filament in between is **not** compensated.
4. If `MOVE=1`: moves X, Y, Z (not E) back to the saved position at `MOVE_SPEED`.

`G61` is otherwise identical to `RESTORE_GCODE_STATE`.

### Syntax

```
RESTORE_GCODE_STATE [NAME=<name>] [MOVE=<0|1>] [MOVE_SPEED=<mm/s>]
G61 [NAME=<name>] [MOVE=<0|1>] [MOVE_SPEED=<mm/s>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `NAME` | Name of the snapshot to restore. | name used when saving | `default` |
| `MOVE` | Also move back to the saved X, Y, Z. | 0 = no, non-zero = yes | 0 |
| `MOVE_SPEED` | Speed of that move. **Unit: mm/s.** | mm/s | the speed in force after the restore (the saved speed) |

### Examples

```
RESTORE_GCODE_STATE NAME=before_wipe MOVE=1 MOVE_SPEED=100
```
Return to the saved position at 100 mm/s and restore the modes.

```
RESTORE_GCODE_STATE
```
Restore the `default` snapshot without moving.

### Notes and cautions

- Without `MOVE=1` the **modes** (e.g. absolute) are restored but the toolhead stays where it is: the next absolute move goes from here to the given coordinates.
- The state saved by `PAUSE` is restored by `RESUME` (chapter 21).

---

## `M220` — Speed factor

### What it does

Sets a **percentage** that multiplies the feed rate (`F`) of all moves, from now on. The current speed is rescaled at once, so the change is immediate for the moves that follow without a new `F`.

- `M220 S100` = normal; `S50` = half speed; `S200` = double speed.
- The value is a percentage of the **programmed** speed, **not** a limit: the printer's own limits (`max_velocity`, acceleration, see chapter 28) still apply, so S200 may give less than twice the speed.

### Syntax

```
M220 [S<percent>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `S` | Speed factor in percent. | greater than 0 | 100 |

### Examples

```
M220 S80
```
Slow down all moves to 80 %.

```
M220 S100
```
Back to normal.

### Notes and cautions

- `S0` or a negative value gives an error: **`Error on 'M220 S0': S must have above of 0.000000`**; the factor is not changed.
- `M220` without `S` sets 100 %.
- The same factor can be changed from the touchscreen/web interface; the last one wins. `M222` also resets it.
- It is saved/restored by `SAVE_GCODE_STATE`/`RESTORE_GCODE_STATE`.

---

## `M221` — Extrusion (flow) factor

### What it does

Sets a **percentage** that multiplies the `E` of all following moves (arcs included): `M221 S95` pushes 95 % of the filament the slicer asked for.

The extruder's G-code position is adjusted at the same time so that the position **does not jump**: only the moves after the command are affected.

### Syntax

```
M221 [S<percent>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `S` | Extrusion factor in percent. | 100 = normal | 100 |

### Examples

```
M221 S105
```
5 % more filament from now on (slightly over-extruding material).

```
M221 S100
```
Normal flow.

### Notes and cautions

- Unlike `M220` **there is no check on `S`**: `S0` and negative values are accepted (verified: no limit is passed to the parameter reader).
- **Do not use `S0`.** With a factor of 0 every `E` becomes 0 (the filament does not advance), and the next `M221` (any value) divides the extruder position by the old factor, 0, which gives "not a number" (NaN) — the computation does not stop on a division by zero. The extruder's G-code position is then NaN, and later extruding moves inherit it; `G92 E0` may or may not repair it (it works only as long as no extruding move has been made in between), so the safe recovery is to switch the printer off and on again. The NaN is derived from the arithmetic of the code; what the motion planner does with a NaN extruder move was not examined.
- Applies to `G92 E` values and to `M114`'s `E` as well (they are shown/entered in "slicer" filament units).
- Saved/restored by `SAVE_GCODE_STATE`/`RESTORE_GCODE_STATE`.

---

## `M222` — Reset speed, factors and modes

### What it does

Anycubic's reset command, no parameters:

1. Speed (the remembered `F`) = **25 mm/s** (equal to `F1500`).
2. Speed factor (`M220`) = **100 %**.
3. Extrusion factor (`M221`) = **100 %**.
4. Speed mode number = **1**. This is the touchscreen's *speed mode*. The interface knows three modes with these limits (from the printer's interface configuration): **stable = 50**, **standard = 100**, **sport = 130** (percent of the programmed speed). Mode 1 matches "standard" with a 100 % factor: the live printer reports `speed_mode: 1` with `speed_factor: 1` (100 %). That numbers 0 and 2 are stable and sport is inferred from the order of the names (to verify). The interface sets the speed through the factor; in the firmware the mode number is stored, saved/restored with the G-code state and reported to the interface. No other use of it was found there (not exhaustively scanned).
5. Absolute positioning of the axes (`G90`) and of the extruder (`M82`).

It does **not** move anything and does not change offsets (`G92`, `SET_GCODE_OFFSET`), positions or saved states.

### Syntax

```
M222
```

### Parameters

None.

### Examples

```
M222
```
Put the printer back to a clean motion state before a manual sequence.

### Notes and cautions

- Neither the sample G-code files nor the stock `printer.cfg` use it.
- The speed setting of the touchscreen/web interface works through a separate request (a speed percentage plus a "mode" number), not through G-code; it changes the same speed factor, so `M220` and the interface override each other.

---

*End of chapter 19.*

---

# Chapter 20 — OTA (firmware update of a control board)

One command, `OTA_START`. It does **not** update the printer's main firmware (that is done through the update package). It pushes a new firmware file into one of the printer's **microcontroller boards** over the serial link: the main board (`[mcu]`) or the nozzle board (`[mcu nozzle_mcu]`).

| Command | Purpose |
|---|---|
| `OTA_START` | Send a firmware file to a control board and let the board flash itself |

(The separate command for the filament hub, `OTA_FILAMENT_HUB_START`, is in chapter 1.)

---

## `OTA_START` — Update the firmware of a control board

### What it does

1. Takes the **file name** (the folder part is ignored) and checks that it looks like `firmware_vX.Y.Z_YYYYMMDD.bin`, where X, Y, Z are numbers of 1 to 3 digits and the date has 8 digits (example: `firmware_v1.2.3_20250115.bin`). The numbers X.Y.Z become the **version** of the new firmware. If the name does not match, the command stops with the error `firmware file format error`.
2. Opens the file and reads it completely. If this fails, the update is marked as failed (`open file error` / `read file error`) and the command returns an error.
3. Computes a checksum (CRC32) of the file **without its last 9 bytes**. (The file must therefore be longer than 9 bytes. In the firmware file shipped in the update package, `firmware_v1.4.2_20250703.bin`, the last 9 bytes are the text `c37d560d` followed by a line feed — 8 hexadecimal characters that look like a build identifier, appended after the actual program code, which starts with a standard ARM microcontroller vector table. They are simply left out of the checksum.)
4. Asks the selected board which firmware it currently runs (version and checksum), if not already known.
5. **If the board already has the same version and the same checksum**, nothing is done: the update is logged as `not_need_upgrade`, the command ends without error and nothing is sent. Otherwise the new version is written to a small version file kept by the firmware.
6. Starts a watchdog timer (10 s) and tells the board "an update with this checksum and this version is coming", then asks it to erase its update area.
7. The board then **requests the file piece by piece**; the printer reads the requested part of the file and sends it. Progress runs from 0 to 100 %. Each piece restarts a 5 s watchdog.
8. When the board has received, checked and written everything (`write finish`), it **restarts itself** with the new firmware. The command waits for this (it polls every 0.1 s) and ends without error. A success event is then raised and the stored version is updated.

During the update the command **blocks** the G-code queue until the board is done or an error happened.

### Syntax

```
OTA_START MCU=<board> UPDATE_PATH=<file>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `MCU` | Which board to update. The command is registered once per board; the value is the board's name from the configuration. | `mcu` (main board), `nozzle_mcu` (nozzle board): the name of the `[mcu …]` section in `printer.cfg`, without the word `mcu` for the main one (standard Klipper naming for multi-board commands; the command is registered with these names, but was not tried on the printer) | none (required) |
| `UPDATE_PATH` | Path of the firmware file, on the printer's file system (relative paths are relative to the program's folder). The file name must follow `firmware_vX.Y.Z_YYYYMMDD.bin`. | text | if omitted, the firmware uses the built-in example name `./firmware_v0.1.1_20230712.bin` (the same text as in the help; no such file exists on the printer, so the command then fails with an open-file error) |

### Examples

```
OTA_START MCU=nozzle_mcu UPDATE_PATH=/tmp/firmware_v1.2.3_20250115.bin
```
Update the nozzle board from a file copied into `/tmp`.

```
OTA_START MCU=mcu UPDATE_PATH=/useremain/firmware_v1.0.9_20241201.bin
```
Update the main board.

### Errors you may see

| Message | Cause |
|---|---|
| `firmware file format error` | file name does not match the pattern |
| `open file error`, `read file error` | file missing or unreadable |
| `ota transfer timeout` | the board stopped asking for data for about 5 s |
| `MCU responds to ota transfer finish timeout` | the board did not answer after the last piece |
| `wait for mcu erase flash timeout` | the board did not confirm the erase |
| `ota data check error`, `ota data write error`, `flash write error`, `flash erase error` | the board reported a failure while checking or writing |

### Notes and cautions

- **Dangerous command.** A failed or interrupted update of a control board can leave the printer unusable (the board may need to be re-flashed by hand). Never run it during a print, never cut the power, and only with a file made for this board by Anycubic.
- **Most failures shut the printer down.** In this firmware, errors that the command *returns* (as opposed to messages only shown in the console) put the printer in shutdown ("Internal error on command"), so the printer must be **switched off and on again** (power cycle) afterwards (`FIRMWARE_RESTART` is not a reliable way out on this printer; inferred from the general error handling, not tested). The "same version, nothing to do" case is the exception: it ends quietly.
- The firmware does not check that the file really belongs to the selected board, nor the heater/print state; it only checks the file name and the board's own integrity checks (size limits of the update area come from `fw_max_size` 40960, `fw_sector_size` 2048 and `fw_ota_sector_offset` in the board's configuration section).
- The file name regex is not anchored: a name such as `x_firmware_v1.2.3_20250115.bin_old` is also accepted (the pattern has no start/end anchor; only the file name, not the folder, is tested).
- Version comparison uses the version number and the checksum; to force a reflash of the same version the file content has to differ.
- A web/touchscreen interface request `ota/query_version` returns the current board version and update progress; the update status words are `ota start`, `upgrading_download`, `writing`, `transfer_finish`, `upgrading_restart`, `error`.
- **Not test-run.** The command was only analysed, never run. Nothing in the printer's own scripts or configuration calls it; the shipped firmware file `firmware_v1.4.2_20250703.bin` (36 249 bytes, below the 40 960 byte limit) is not referenced by name anywhere that was found, so which board it targets is unknown.

---

*End of chapter 20.*

---

# Chapter 21 — Pause, resume and cancel

Seven commands control a print that is running. They share one "paused" state of the printer; most of the real work (parking, cooling, re-priming) is done by background listeners that the commands trigger.

| Command | Purpose |
|---|---|
| `PAUSE` | Pause the print: lift, retract, park, nozzle to 140 °C |
| `M600` | Exactly the same as `PAUSE` (filament change) |
| `M601` | Exactly the same as `PAUSE` (the Anycubic slicer's pause G-code) |
| `RESUME` | Reheat, re-prime, go back to the saved position and continue |
| `CANCEL_PRINT` | Stop the print and run the clean-up (heaters off, lift, park, motors off) |
| `SET_PAUSE` | Only mark the printer as "paused" (no movement) |
| `CLEAR_PAUSE` | Only remove the "paused" mark (no movement, nothing resumes) |

`M600`/`M601` have **no parameters** (no `X Y Z E L`, no message): anything written after them is ignored. There is **no `M25`** in this firmware (see chapter 22 for the print-job commands).

Settings used (section `[pause_resume]` of `printer.cfg`, and the filament-hub section `[filament_hub]`):

| Setting | Stock value | Used for |
|---|---|---|
| `pause_resume` → `pause_position` | `457,420` | Park position; **only the X value (457) is used** by the pause |
| `pause_resume` → `recover_velocity` | not in stock file, default **50** mm/s | Speed of the return move of `RESUME` when `VELOCITY` is not given |
| `filament_hub` → `pre_sweep_position` / `sweep_position` | 457 / 445 | The X positions between which the head wipes before re-priming |
| `filament_hub` → `cruising_speed` | 200 mm/s | Speed of the long travel moves of pause/resume |
| `filament_hub` → `delay_before_throw` | 3 s | Pause after the re-priming (see `RESUME`) |
| `filament_hub` → `sweep_fan_speed` | not in stock file, default 0 | Fan speed during that short pause (0 = fan off) |

(`z_up: 10`, `extru_pull: 4` and `bee_path` in `[pause_resume]` are **not used** by these commands; the lift of 10 mm and the retraction of 2 mm are fixed values in the firmware.)

---

## `PAUSE` / `M600` / `M601` — Pause the print

### What it does

All three names run the same code.

1. If the printer is **already paused**, it only prints `Print already paused` and does nothing else.
2. Otherwise it stops the print job from reading further lines (a line already being executed finishes) and, if no file is being printed, announces the pause to the interface instead (`action:paused`).
3. It saves the current G-code state under the name `PAUSE_STATE` (like `SAVE_GCODE_STATE NAME=PAUSE_STATE`, chapter 19): position, speed, speed factor, extrusion factor, absolute/relative modes.
4. It marks the printer as paused and sets the print state to **paused**.
5. It then triggers the pause listener, which performs the physical pause (in this order):
   1. remembers the **fan speed**, the **nozzle target temperature** and the **X/Y/Z position**;
   2. switches the **part-cooling fan off**;
   3. sets the nozzle target to **140 °C** — it does **not** wait for the nozzle to cool; the bed stays at its temperature;
   4. if X, Y and Z are all homed: (a) when the file is printed *object by object* and it declares its tallest height (`; max_z_height:` line), the head first goes up to that height at 30 mm/s so it clears every object; (b) **retracts 2 mm** of filament at 30 mm/s; (c) **lifts Z by 10 mm** at 30 mm/s; (d) **moves X to 457** (the `pause_position` X value) at 200 mm/s — Y is not changed;
   5. waits until all these moves are finished.
   If the axes are not all homed, steps (a)–(d) are skipped (the fan and the temperature change still happen).
   (With the ACE multicolour hub connected the listener also stops the feed assist and activates the RFID reader; not detailed.)

Nothing is printed to the console on success.

### Syntax

```
PAUSE
M600
M601
```

### Parameters

None.

### Examples

```
M600
```
In a G-code file, at the layer where you want to change filament. The nozzle parks on the right, cooled to 140 °C.

```
M601
```
What the Anycubic slicer inserts for its "pause at layer" function (`machine_pause_gcode = M601`).

### Notes and cautions

- **A filament change is not automated.** There is no unload or load: only a 2 mm retraction. You unload and load the new filament yourself while parked (use the touchscreen or `EXTRUDE_FILAMENT`-type functions); the nozzle is at 140 °C, which is too cold to push most filaments — the heating back up happens only in `RESUME`.
- The paused state does not time out in anything that was found. Leaving the nozzle at 140 °C is safe for the nozzle, but filament left sitting in it may ooze or degrade.
- If the pause is sent while the printer is **not printing a file** (for example from the console), the same movements are made if the axes are homed; `RESUME` then restores the saved position.
- **Pause outside a print.** The filament hub keeps a state number (0 idle, 1 printing, 2 paused). It is set to 1 when a print file starts, after a power-loss recovery and after a resume; a print end or cancel sets it to 0. `PAUSE` always does the fan, temperature and movements, but only switches the hub to "paused" when it was "printing". When the hub was **not** "printing" (for example `PAUSE` typed in the console with no print), `RESUME` still reheats the nozzle (`M109`) and then restores the saved state, but **skips the wipe, the 80 mm purge, the delay and the separate X/Z return** (verified in the code; not tested on the printer).

---

## `RESUME` — Continue after a pause

### What it does

1. If the printer is **not paused**: prints `Resume aborted: print is not paused` and stops.
2. Triggers the re-start listener, which does, in this order (standalone spool, **no ACE hub**):
   1. if a nozzle temperature was remembered at the pause: the head wipes back and forth between X 445 and X 457 (long moves at 200 mm/s, wipes at 20 mm/s) — this is done for the Kobra 3, 3 V2, 3 Max and 3 Max V2 — then **`M109`** heats the nozzle back to the remembered temperature and **waits for it**;
   2. **extrudes 80 mm of filament at 5 mm/s** (about 16 s) to purge the old colour/air and refill the nozzle, and waits for the move to end;
   3. if `delay_before_throw` is above 0 (3 s stock): sets the fan to `sweep_fan_speed` (0 = off, stock) and **waits that many seconds** (lets the strand drop);
   4. restores the **fan speed** it had before the pause;
   5. moves X back to its pre-pause X (at 200 mm/s) and then Z back to its pre-pause Z (at 10 mm/s).
3. Restores the saved state with `RESTORE_GCODE_STATE NAME=PAUSE_STATE MOVE=1 MOVE_SPEED=<speed>` (chapter 19): returns to the exact saved X/Y/Z position and gives back speed, factors and modes. Because of the rule described in chapter 19, the extruder's G-code position is put back to the value it had at the pause, so the 80 mm that were purged do not count as printed filament (inferred from that rule, not tested).
4. Restarts the reading of the print file, clears the paused mark, and the print continues.

### Syntax

```
RESUME [VELOCITY=<mm/s>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `VELOCITY` | Speed of the return move to the saved position (step 3), in **mm/s**. | number > 0 | the `recover_velocity` setting, **50** mm/s when not set |

### Examples

```
RESUME
```
Continue after you changed the filament.

```
RESUME VELOCITY=20
```
Same, with a slower return to the print position (useful if the nozzle has to pass close to the part).

### Notes and cautions

- Before sending `RESUME`, **load the new filament up to the nozzle**: the re-priming only pushes 80 mm at 5 mm/s without checking anything. If the filament is not in the extruder gears, nothing comes out.
- The nozzle returns from above (Z +10 mm was kept until the final move), but check that nothing (stringing, a blob) is in the way of the return move.
- With the ACE hub connected the sequence is different (the hub handles purge and feed); not detailed here since it is not used.
- **To verify:** the 80 mm / 5 mm/s values and the order of the steps are read from the code; they were not timed on a real pause. Whether the `VELOCITY` value is accepted as 0 or negative was not examined.

---

## `CANCEL_PRINT` — Cancel the print

### What it does

1. If a print file is active (or the printer is paused), it **cancels** the print job: stops the reader, closes the file and runs the clean-up below. Otherwise this step is skipped.
2. Clears the paused marks.
3. Deletes the **power-loss recovery record** of the print (so the printer will not offer to resume this print after a power cut).

The clean-up of step 1 does, in order:

1. runs `G9113` (nozzle-clean counter/state, chapter 2);
2. resets speed to 25 mm/s, speed factor 100 %, extrusion factor 100 %, speed mode 1, absolute positioning (`G90`) and absolute extrusion (`M82`) — the same as `M222` (chapter 19);
3. resets the velocity limits to the configured ones (`RESET_VELOCITY_LIMIT`, chapter 28) and sets the extruder position to 0 (`G92 E0`);
4. **switches off all heaters and the fan**;
5. waits until the current moves end; **if X, Y, Z are homed**: when the file was printed object by object, goes up to the tallest height (`max_z_height`) at 30 mm/s; then **lifts Z by 15 mm** (from the current height) at 15 mm/s; then runs the `MOVE_END_POS` macro (X 0, Y 420 — see chapter 16);
6. **turns the motors off** (like `M84`, so the axes become "not homed");
7. re-enables the stealth mode of the X and Y motors (`OPEN_STEALTHCHOP`, chapter 26) and sets `M900 K0.02`;
8. deletes the power-loss record and forgets the adaptive-mesh information of the file.

If the cancel comes from the **touchscreen or web interface** (not from G-code), the interface first prints `cancel_request:stopping`, runs `TURN_OFF_HEATERS`, and only then runs `CANCEL_PRINT`; when the printer state is a calibration (flow calibration, resonance test, leveling), the interface runs `CANCEL_PRINT` directly, at once, inside whatever is running; in every other state it goes through the normal G-code queue (waits for the command in progress, and raises the internal "before/after cancel" events). The cancel itself is the same in both cases. Either way the interface finishes by signalling the end of the cancel (so the interface can reset its own state).

### Syntax

```
CANCEL_PRINT
```

### Parameters

None.

### Examples

```
CANCEL_PRINT
```
Abort the print from the console. The heaters turn off, the head lifts 15 mm and goes to X 0 / Y 420, then the motors are released.

### Notes and cautions

- Because the motors are released, **the printer must be homed again** (`G28`) before any other movement.
- The cancel is irreversible: the recovery record is deleted, so a print cancelled this way cannot be resumed.
- With the multicolour hub the cancel also retracts and unwinds the filament (not detailed).
- `M900 K0.02` puts the pressure advance back to the stock value 0.02 after every cancel (the same value as `pressure_advance` in `printer.cfg`); at the same point the result of any `PA_AUTO_CALIBRATE` is erased, so the `K` ignore rule of chapter 7 stops applying (see chapters 4 and 7). A `K` value set in your own G-code is therefore lost at the cancel.

---

## `SET_PAUSE` — Mark the printer as paused

### What it does

Only sets the two internal "paused" flags to true. **No move, no state saving, no print-job pause.** It exists so other code (for example a filament-runout handler) can declare that the printer is paused; without a saved `PAUSE_STATE`, a later `RESUME` would restore whatever state was saved before.

### Syntax / parameters

```
SET_PAUSE
```
No parameters.

### Notes and cautions

- For a normal pause use `PAUSE`. Using `SET_PAUSE` by hand makes `RESUME` possible and `PAUSE` answer "already paused", but nothing is actually stopped.

---

## `CLEAR_PAUSE` — Remove the "paused" mark

### What it does

Clears the two "paused" flags without resuming anything and without moving. The `G9111` print-start macro (chapter 2) runs it at the beginning of each print so that a forgotten pause cannot affect the new print.

### Syntax / parameters

```
CLEAR_PAUSE
```
No parameters.

### Notes and cautions

- It does **not** restart a print job that was paused, and does not reheat the nozzle. Use `RESUME` for that.
- If the print was paused by `PAUSE` and you use `CLEAR_PAUSE`, the print stays stopped but the printer believes it is not paused: `RESUME` then answers "print is not paused". Use `CANCEL_PRINT` to get out.

---

*End of chapter 21.*


---

# Chapter 22 — Print job (virtual SD card): `SDCARD_PRINT_FILE`, `SDCARD_RESET_FILE`, `M73`, power-loss recovery, `SLEEP_MODE`

These commands deal with the *print job itself*: loading a G-code file and starting it, forgetting it, telling the screen how far the print has progressed, and the power-loss recovery ("continue after power cut"). The job reader is called the "virtual SD card"; in `printer.cfg` it is the `[virtual_sdcard]` section (stock: `path: /useremain/app/gk/gcodes/` and `enable_continue_print: 1`).

| Command | Purpose |
|---|---|
| `SDCARD_PRINT_FILE` | Load a `.gcode` file and start printing it |
| `SDCARD_RESET_FILE` | Forget the loaded file (stops reading it); heaters and motors are not touched |
| `M73` | Report progress and remaining time (written by the slicer into the file, shown on the screen) |
| `SET_RESTORE_POWER_OFF` | Switch the power-loss recovery on/off while the printer runs |
| `SDCARD_RESUME_PRINT` | Resume a print interrupted by a power cut (`ENABLE=1`), or discard the saved record (`ENABLE=0`) |
| `SDCARD_OPEN_PRINT_PREV_OP` | Ask for the three pre-print operations (auto-level, vibration/resonance, flow calibration) for the next print |
| `SLEEP_MODE` | Motors off, fan off, heaters off |

There is **no** `M20`–`M30`, `M23`, `M24`, `M25`, `M26`, `M27` (SD-card commands) in this firmware: only the commands above exist. Pausing is chapter 21.

**Note on errors in this chapter.** Several commands below answer wrong use with an error that the firmware *returns* (as opposed to a console-only message). According to the rule found for chapters 15–16, such an error puts the printer into **shutdown**, which on this printer is recovered by **switching it off and on** (not `FIRMWARE_RESTART`). This is inferred from the general rule, not tested; it is the reason the cautions below say "do not".

---

## `SDCARD_PRINT_FILE` — Load a file and start printing

### What it does

1. **If the SD reader is already busy** (a job is running or a read timer exists): writes a warning in the log (`SD busy detected, forcing cleanup`), then drops the old job exactly like `SDCARD_RESET_FILE` does (see below) before continuing. It does **not** turn heaters off or park the head, so sending this during a print replaces the job under a hot nozzle. Do not do it.
2. Reads the file name (`FILENAME`, required) and loads it:
   - Quote characters `"` around the name are removed.
   - The name **must end in `.gcode`** (lower case) — otherwise the file is refused (message in the log: `Gcode file extension incorrect`).
   - A name **without `/`** is looked up in the folder of the `[virtual_sdcard]` `path` setting (stock `/useremain/app/gk/gcodes/`).
   - A name **containing `/`** is, from the code, joined to the root folder `/`, i.e. treated as a path from the root of the printer's file system, not as a path relative to the G-code folder (to verify — the help text, copied from Klipper, claims "subdirectories").
   - The file must exist (`Unable to open file`) and must not be empty (`Gcode file is empty`).
3. Reads the file's header lines and its last lines (slicer name, layer height, temperatures, `; max_z_height:`, model size, nozzle diameter, `by object` marker, speed meta data, …). This information feeds the status shown on the screen (see `M73` and the status fields at the end of this section).
4. **Checks the file against the machine** — a failure stops the print and the message goes to the screen/log:
   - the nozzle diameter declared in the header (`; nozzle_diameter = 0.4`; first value if a list) must be **exactly equal** to the printer's `nozzle_diameter` of `[extruder]` (stock 0.4) — checked first, only when the header gives one (verified);
   - **only if the file has the four lines `; print_bed_min =`, `; print_bed_max =`, `; first_layer_print_min =`, `; first_layer_print_max =`** (the lines the Anycubic slicer writes, chapter 3): these X/Y values must not exceed the axis maximum (`position_max` of `stepper_x` / `stepper_y`). What is really compared (verified in the code): for **X** the X of `print_bed_max` and the X of `first_layer_print_min`; for **Y** the Y of `first_layer_print_min` and the Y of `first_layer_print_max`. This is an odd mixture (the X of `first_layer_print_max` is never checked), most likely a slip in the firmware, but harmless with normal files (e.g. `print_bed_max = 426,420` against X 457);
   - the model height (header `; model_size`) and `; max_z_height:` must not exceed the Z axis maximum (`position_max` of `stepper_z`).
5. If a filament hub (ACE) is configured, asks it to check that the colours/spools the file needs are available (`check_ready`); if it says no, the print is stopped. It also reads the file in the background to learn which tools `T…` and which first-layer area (adaptive mesh, chapter 3) it uses.
6. Looks at the saved configuration (`printer_mutable.cfg`): if it contains **no `[input_shaper]` section** (no shaper calibration was ever saved), a log error is written (`The device is no resonance…`) and the job gets the **"resonance" pre-print operation** switched on (see `SDCARD_OPEN_PRINT_PREV_OP` below: nozzle and bed are started heating and the state shows `resonance` before the file's own lines run). On a printer that has a saved shaper this does not occur.
7. Starts the job: the file is read line by line from the beginning and executed. State becomes `printing`.

**When something is refused (steps 2, 4, 5)** the firmware does not simply reject: it runs its **cancel routine** (events `reset_file` and `cancel_print`, then the clean-up of chapter 21's `CANCEL_PRINT`: heaters off, fan off, lift, motors off…). The command itself then ends without an error. So a mistyped file name on a hot printer also switches the heaters off and un-homes the printer.

### Syntax

```
SDCARD_PRINT_FILE FILENAME=<name.gcode>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `FILENAME` | The G-code file to print. | A name ending in `.gcode`, with or without quotes; a name with `/` is taken from the file-system root (see above) | none — required (missing → error `Error on 'SDCARD_PRINT_FILE': missing FILENAME`, shown on the console only) |

### Examples

```
SDCARD_PRINT_FILE FILENAME=cube.gcode
```
Prints `cube.gcode` from the stock G-code folder.

```
SDCARD_PRINT_FILE FILENAME="my part.gcode"
```
The quotes allow a space in the name (the quotes are removed).

### Notes and cautions

- The file's own start code (for the Anycubic slicer `G9111`, chapter 2, then `M117` — see chapter 5) does the heating, homing and mesh. `SDCARD_PRINT_FILE` itself heats nothing.
- Files with an upper-case extension (`.GCODE`) or another extension are refused, and the refusal triggers the cancel routine described above.
- Status fields filled from the file (read with the Moonraker `virtual_sdcard` object, for example by Mainsail): `progress`, `file_position`, `file_size`, `total_time`, `remain_time`, `time_left`, `total_layer`, `current_layer`, `layer_height`, `filament_type`, `bed_temperature`, `extruder_temperature`, `model_size_x/y/z`, `slicer`, … They are taken from the slicer's comment lines; a file from a slicer that does not write them leaves them at their default (0 or −1).
- At each layer change comment (`;LAYER:` or `;LAYER_CHANGE`) the print counts one more layer, and — with the power-loss recovery on — saves a recovery record (see `SDCARD_RESUME_PRINT`).

---

## `SDCARD_RESET_FILE` — Forget the loaded file

### What it does

1. **If the command is itself executed from the print file** (a line of the G-code file calls it): refuses with the error `SDCARD_RESET_FILE cannot be run from the sdcard`. This error is returned, so it is a shutdown-type error (inferred, see the note at the top).
2. Otherwise: if a file is open, stops the reading (the print job timer is stopped) and closes the file.
3. Clears **everything the firmware knew about the job**: file position and size, progress, remaining times (set back to "unknown"), layer counters, slicer information, filament and temperature information, model size, speed meta data, adaptive-mesh data, the calibration marks.
4. Sends the event `printer_macro:reset` (listeners such as the print statistics see the job as gone).

It does **not** turn heaters or fan off, does not move anything and does not run the cancel routine (compare `CANCEL_PRINT`, chapter 21). Whether the print-state text ("printing"/"paused"…) is changed by it was not checked (to verify).

### Syntax

```
SDCARD_RESET_FILE
```

### Parameters

None.

### Examples

```
SDCARD_RESET_FILE
```
From the console, after `CANCEL_PRINT`, to clear a stuck "file loaded" display.

### Notes and cautions

- Use `CANCEL_PRINT` to end a print properly; `SDCARD_RESET_FILE` is only the bookkeeping part.
- Do not put it in a G-code file (error above).

---

## `M73` — Progress and remaining time

### What it does

Stores the values given and nothing else (no move, no heating, no message). The Anycubic slicer writes a line such as

```
M73 P12 R7.5 QUIETR14.3 SPORTR5.2
```

at every percent of the print. The values are used by the status that the screen and Mainsail show:

- **`P`** → the progress of the print in %, shown as `progress` (a fraction 0–1).
- **`R`**, **`QUIETR`**, **`SPORTR`** → the remaining time **in minutes** at the *standard*, *quiet* and *sport* speed modes (the speed mode is the one chosen on the touchscreen, chapter 19, `M222`/`set_velocity`). They are stored in seconds (×60).

How the status is derived:

- `progress`: the value of `P` if one was received, otherwise file position ÷ file size.
- `remain_time`: the value stored for the **current speed mode** (quiet = `QUIETR`, standard = `R`, sport = `SPORTR`). If that value is "unknown" (−1, which is the case after loading a file and before the first `M73`), it is computed as (1 − progress) × `total_time`, where `total_time` is the estimated print time from the file header.
- `time_left`: a second remaining-time field, chosen from the current speed factor (≤ 80 % → `QUIETR`, ≥ 120 % → `SPORTR`, between → another stored value), else the same fallback computation, else −1. Its values are multiplied by 60 once more than `remain_time`, so its unit looks inconsistent (to verify; `remain_time` is the field to use).

### Syntax

```
M73 [P<percent>] [R<minutes>] [QUIETR<minutes>] [SPORTR<minutes>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `P` | Progress in %. | 0 to 100 (outside this range → console error, nothing stored) | −1 = not given (progress is then computed from the file position) |
| `R` | Remaining time at standard speed, in minutes. | number | −1 → see warning below |
| `QUIETR` | Remaining time at quiet speed, in minutes. | number | −1 → see warning below |
| `SPORTR` | Remaining time at sport speed, in minutes. | number | −1 → see warning below |

### Examples

```
M73 P50 R30 QUIETR38 SPORTR26
```
Progress 50 %, 30 min remaining at standard speed, 38 at quiet, 26 at sport.

### Notes and cautions

- **Every call overwrites all four values.** A parameter that is left out is not "kept": `P` goes back to −1 (progress then follows the file position) and, from the code, `R`, `QUIETR` and `SPORTR` are stored as −1 **×60 = −60** s, which is not recognised as "unknown" — so a call like `M73 P50` alone would show a remaining time of −60 s (to verify; a slicer always gives all four).
- The values are lost when the file is reset/cancelled (they return to −1).
- A file without `M73` lines (other slicers) still shows `progress` from the file position, and `remain_time` from the header estimate if it has one.

---

## `SET_RESTORE_POWER_OFF` — Power-loss recovery on/off

### What it does

Stores the number given in the "power-loss recovery" setting, **in memory only** (it is not written to `printer.cfg` and is read again from `enable_continue_print` at the next start). Nothing is printed.

The setting has two roles:

- while it is **exactly 1**, the print saves a recovery record at every layer change (from the second layer on) — see `SDCARD_RESUME_PRINT`;
- `SDCARD_RESUME_PRINT` refuses to run when it is **0**.

### Syntax

```
SET_RESTORE_POWER_OFF [ENABLE=<n>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `ENABLE` | New state. | `1` = on; `0` = off. Any other whole number is stored but behaves as a half-on state (recovery is allowed but no record is saved) — do not use. | `0` |

### Examples

```
SET_RESTORE_POWER_OFF ENABLE=0
```
Stop saving recovery records for the rest of the session (for example for a test print).

```
SET_RESTORE_POWER_OFF ENABLE=1
```
Switch it back on. (A bare `SET_RESTORE_POWER_OFF` switches it **off**.)

### Notes and cautions

- To make the choice permanent, change `enable_continue_print` in `[virtual_sdcard]` (stock `1`). The firmware reads it as a whole number.
- Switching it off does not delete an existing record (use `SDCARD_RESUME_PRINT ENABLE=0` for that — and it refuses while the setting is 0, so delete first).

---

## `SDCARD_RESUME_PRINT` — Continue after a power cut

### What it does

While power-loss recovery is on, the printer writes a small record every layer (numbered files, 11 of them in rotation, so a record is never half-written). The record holds the G-code state (`SAVE_GCODE_STATE NAME=POWER_OFF`: position, speeds, factors, modes), the file name and position, the nozzle and bed targets, the fan speed, the progress values and the elapsed print time. At the end of a normal print and at a cancel the records are deleted.

After a power cut the screen finds the record (it answers `Print record does not exist` when there is none) and offers to continue. The command does the work:

1. **If the recovery setting is 0:** error `The power outage function has been turned off.` (returned error → shutdown type, inferred).
2. **If a print is running:** error `printing` (same remark).
3. `ENABLE` is **required**:
   - **`ENABLE=0` (or anything but 1): discard.** The saved records are deleted and nothing else happens.
   - **`ENABLE=1`: resume.** The command reads the records, takes the most recent valid one, reopens the G-code file at the saved position, restores the exclude-object list, and starts the recovery sequence, which runs by itself:
     1. nozzle target **185 °C** (no wait) and the saved **bed** target (no wait), so heating starts at once;
     2. homes **X and Y only** — `G28 X Y` (Kobra 3 / 3 V2) or `HOME_XY` (Max and others, see chapter 12) — and sets the motors to the non-silent mode (`CLOSE_STEALTHCHOP`, chapter 26) for X, Y (and Y1 on the Max); **Z is not homed** — the printer reads the last Z height it had written to a small file (`last_z`, kept while printing) and declares the toolhead to be at that height without moving (an internal helper; exact details to verify). It also reapplies the saved first-layer speed settings and the absolute/relative extruder mode;
     3. waits 0.1 s;
     4. heats the nozzle to the saved temperature and **waits** (`M109`), then the bed (`M190`) and waits;
     5. (non-ACE printers) `G92 E0`, then **`G1 E100 F300`: extrudes 100 mm at 5 mm/s** (about 20 s) to refill the nozzle, waits for the move, then `G4 P3000` (3 s) and turns the **part-cooling fan on at full speed** (the fan routine takes 0–255 and 255 = 100 %, verified; the fan speed saved in the record is not set here);
     6. `RESTORE_GCODE_STATE NAME=POWER_OFF MOVE=1 MOVE_SPEED=50` — goes back to the saved X/Y/Z position at **50 mm/s** and restores speeds and modes (chapter 19);
     7. marks the print as `printing`, restarts the print statistics and starts reading the file from the saved line.
   - With the ACE hub in use, steps 5–6 are done by the filament hub instead (not detailed).

### Syntax

```
SDCARD_RESUME_PRINT ENABLE=<0|1>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `ENABLE` | `1` = resume the interrupted print; anything else (use `0`) = delete the saved record without printing. | whole number | none — required (missing → console error `Error on 'SDCARD_RESUME_PRINT': missing ENABLE`) |

### Examples

```
SDCARD_RESUME_PRINT ENABLE=1
```
Continue the interrupted print (the screen sends this when you press "Continue").

```
SDCARD_RESUME_PRINT ENABLE=0
```
Throw the saved record away ("Cancel" on the recovery dialog).

### Notes and cautions

- **Z is not re-homed.** The print continues at the Z the printer thinks it had at the cut. If the bed or the head was moved by hand after the cut (or the printer lost steps), the continuation will be at the wrong height. Check by eye before pressing continue.
- The recovery is as precise as the layer boundary (inferred): the print restarts from the saved file line, which is the start of the layer in which the cut happened (a record is written at each layer change), so the layer is repeated from its beginning on top of what was already printed (inferred from where the record is written).
- The nozzle is heated to the **saved** temperature, but for the first moments the nozzle target is 185 °C; the 100 mm purge happens at the saved temperature (after `M109`).
- Records exist only from the **second layer**; a power cut during the first layer cannot be resumed.
- A failed command in this chapter that returns an error (the two errors above) should be treated as shutdown-type: power-cycle the printer.

---

## `SDCARD_OPEN_PRINT_PREV_OP` — Calibration-job marks

### What it does

Sets the three **pre-print operation** marks at once: **auto-leveling**, **resonance (vibration) measurement** and **flow (pressure-advance) calibration**. The touchscreen sets them one by one from its print-start options; this command sets all three. Nothing happens when it is run; the marks are used by the **next print** (`SDCARD_PRINT_FILE`). They are processed when the job starts, before the first line of the file is read (verified in the code, the exact effect of each step not tested), and are **cleared right afterwards**, and also when a print ends or is cancelled:

1. If the file header says `; test_mode: 1`, all three marks are dropped (the file declines the operations); `; test_mode: 2` only writes a log error (`AC Slicer test_mode 2 is not support!`). The event `flow_calibration:reset_status` is sent.
2. **Flow calibration mark** — only when an ACE hub is connected: the print state becomes `flow_cali` and `PA_AUTO_CALIBRATE USE_ACE=0` is run (chapter 4); the configuration is saved (`SAVE_CONFIG`) when that routine does not report a failure. Without ACE this mark does nothing.
3. **Resonance mark**: the print state becomes `resonance`; unless the flow mark is also set, the bed is started heating to the file's first-layer bed temperature (`M140`, no wait) and the nozzle to the `[leviQ3] extru_end_temp` value (`M104`, no wait; stock 140 °C). The actual vibration test is **not** started by this code (it is run by the touchscreen separately, to verify).
4. **Leveling mark**: the print state becomes `leveling`; `LEVIQ_AUTO_ZOFFSET_ON_OFF ON_OFF=1` is run (chapter 14), the `[leviQ3] bed_temp_cfg` value is set to the file's bed temperature (a log line `bed temperature mismatch: cfg:… -model:…` appears when they differ) and `LEVIQ B=<bed temperature>` is run (chapter 14: preheat, wipe, probe).
5. The marks are cleared, the state becomes `printing` and the file is read.

A mark is skipped when the print state is already `error`.

### Syntax

```
SDCARD_OPEN_PRINT_PREV_OP
```

### Parameters

None.

### Notes and cautions

- Mostly for the touchscreen; its help text (copied from another command) is wrong. Typing it before `SDCARD_PRINT_FILE` makes that print start with the leveling routine (probing the bed, heating to the file's bed temperature) and, with an ACE hub, a pressure-advance calibration — even if the file's own start code does a leveling too.
- If the file has `; test_mode: 1` the marks are ignored.

---

## `SLEEP_MODE` — Everything off

### What it does

1. Turns **all motors off** (like `M84`, chapter 15: this un-homes X, Y and Z).
2. Sets the part-cooling **fan to 0**.
3. Sets the **target of every heater** (nozzle and bed) to **0** (it also clears an internal state flag of each heater, meaning not known).
4. Only on the **Kobra 3** and **Kobra 3 V2**: sends the event `led_off` (on the Max the event is not sent, so **nothing happens to a light**; inferred from the model test in the code).

It does not wait for anything, does not park the head and prints nothing.

### Syntax

```
SLEEP_MODE
```

### Parameters

None.

### Examples

```
SLEEP_MODE
```
Cool everything down and release the motors in one command, for example at the end of a G-code file instead of `TURN_OFF_HEATERS`, `M107`, `M84`.

### Notes and cautions

- Unlike `TURN_OFF_HEATERS` + `M84`, it does not go through the normal delayed heater/fan queue: the fan and heater targets change immediately.
- The touchscreen's "sleep" function is not this command alone (the screen blanks itself); this only does the machine side.

---

*End of chapter 22.*

---

# Chapter 23 — Print statistics: `M75`, `SET_PRINT_STATS_INFO`

The firmware keeps a small "print statistics" record: file name, print state (`standby`, `printing`, `paused`, `complete`, `cancelled`, `error`), time since the print started, time actually spent printing, filament used, and the layer counters. The touchscreen, Mainsail and Moonraker read it as the object `print_stats`. Two commands act directly on it.

| Command | Purpose |
|---|---|
| `M75` | Restart the "printing time" clock (the print-time counter starts again from zero) |
| `SET_PRINT_STATS_INFO` | Tell the printer the total number of layers and the current layer (for display only) |

Neither command moves anything, heats anything or prints a message.

---

## `M75` — Restart the print-time counter

### What it does

The record keeps two clocks: `total_duration` (time since the job started) and `print_duration` (time spent printing, without the preparation phase and without pauses). `print_duration` is computed as `total_duration − preparation time − time paused`.

`M75` sets the **preparation time** to "everything since the job started". From this moment `print_duration` restarts from about zero. Its help text says "Start the print job timer", which is a little misleading: it does not start the job, it only moves the point where printing time is counted from.

The preparation time is also recomputed by the firmware itself in the background, **as long as no filament has been extruded**: during heating, homing and levelling, `print_duration` stays at zero. After the first extrusion that value is frozen. As a result:

- `M75` **before the first extrusion** has no lasting effect (the firmware recomputes the same value at the next status query).
- `M75` **after the print has started extruding** resets `print_duration` to zero. `total_duration` is **not** changed.

The firmware sends the same reset itself at the start of every print (it is part of what `M117` triggers, see chapter 5), so a normal print never needs `M75`.

### Syntax

```
M75
```

### Parameters

None. Parameters given are ignored.

### Examples

```
M75
```
Restart the printing-time counter, for example at the end of a custom start code to hide a long purge or levelling phase in the time shown on the screen.

### Notes and cautions

- Only the displayed durations change. The remaining time shown on the screen comes from `M73` (chapter 22), not from this counter.
- Sent while **no print is running**, the command sets the preparation time to a meaningless large number (the printer's clock value, since the "start time" is zero). The status can then show a negative `print_duration` until the next print starts (to verify, derived from the code, not tried). Do not send it outside a print.
- Pauses are counted separately: the time paused is not part of `print_duration`, but `M75` forgets the time paused *before* it was sent (it is computed from the job start, not from earlier pauses; derived from the code, not tried).

---

## `SET_PRINT_STATS_INFO` — Total layers and current layer

### What it does

Stores two numbers in the print statistics so that screens can show "layer 12 / 240". The firmware does not compute them itself: the slicer must write the command into the G-code (usually in the start code for `TOTAL_LAYER`, and in the layer-change code for `CURRENT_LAYER`).

The rules, in order:

1. `TOTAL_LAYER` and `CURRENT_LAYER` default to the **values already stored**, so you may send only one of them.
2. If the resulting total is **0**: both stored values are set to **0** (this is how to clear them).
3. Otherwise, if the total is **different from the stored one**: the stored total is replaced and the stored current layer is reset to **0**.
4. Then, if the stored total is not 0, the given current layer is **not 0** and it differs from the stored one: the stored current layer becomes the given value, **limited to the total** (`CURRENT_LAYER=300` with a total of 240 stores 240).

The values appear in the status of `print_stats` as `info: { total_layer, current_layer }` (the live printer shows `info` with `current_layer: 0` and `total_layer: 0` after a file that never sent the command).

### Syntax

```
SET_PRINT_STATS_INFO [TOTAL_LAYER=<n>] [CURRENT_LAYER=<n>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `TOTAL_LAYER` | Number of layers of the whole print | integer, normally ≥ 0; 0 clears both values | the stored value |
| `CURRENT_LAYER` | Layer being printed now | integer, normally ≥ 0; 0 means "leave the stored layer as it is" (it cannot be used to go back to 0, except through `TOTAL_LAYER=0`) | the stored value |

A negative number is **not refused**: the firmware only writes an error line in its log (`Error on 'SET_PRINT_STATS_INFO': TOTAL_LAYER must have minimum of 0`, same for `CURRENT_LAYER`; read from the code, not tried) and then uses the value as given, so a negative layer number would be stored and shown. Do not send negative numbers.

### Examples

```
SET_PRINT_STATS_INFO TOTAL_LAYER=240
```
In the start code: announces a 240-layer print; the current layer is reset to 0.

```
SET_PRINT_STATS_INFO CURRENT_LAYER=12
```
In the layer-change code: layer 12 is now printed.

```
SET_PRINT_STATS_INFO TOTAL_LAYER=0
```
Clear both values.

### Notes and cautions

- The stored layer numbers are **not** the ones shown by the printer's own screen. The firmware keeps a second pair of layer numbers in the print-job reader (`virtual_sdcard` status: `total_layer`, `current_layer`), which comes from the G-code file itself (see "Where the touchscreen gets its layer numbers" below). `SET_PRINT_STATS_INFO` does not influence that pair, and that pair does not feed `print_stats.info`, which is what Mainsail displays.
- Nothing found in the firmware sets these two values by itself and the sample files supplied with the firmware do not send the command, so a display of "layer x / y" works only if the slicer's G-code contains this command (to verify on the screen).
### Where the touchscreen gets its layer numbers, and Orca

Checked on Orca 2.5.0 files stored on the printer and in the firmware. Two independent layer counters exist:

| Counter | Where it appears | Filled by |
|---|---|---|
| `virtual_sdcard.total_layer` / `current_layer` | the printer's touchscreen (it reads this; consistent with what you see) | the file itself, automatically |
| `print_stats.info.total_layer` / `current_layer` | Mainsail (and Moonraker) | the firmware itself: only the `SET_PRINT_STATS_INFO` command; **on a Rinkhals printer, the Rinkhals Moonraker plugin overwrites it with the touchscreen counter** (see below) |

**Touchscreen counter.** When a print is loaded the firmware scans the file (the first lines to find the slicer, then the last lines). The **total** is taken from any of these lines it finds: `; total layers count = N` (Orca writes it in the statistics block at the end of the file), `; total_layers = N`, `;LAYER_COUNT:N` (Cura), or, for the last `; AFTER_LAYER_CHANGE <n> @ <z>mm` line, `n + 1`. The **current layer** starts at 0 and goes up by one each time the printer reads a `;LAYER_CHANGE` or `;LAYER:` comment line while printing. Orca writes `;LAYER_CHANGE` itself at every layer, so nothing has to be configured for the screen. The header line `; total layer number: N` that Orca also writes is **not** read. The last `; AFTER_LAYER_CHANGE` line comes from the *Layer change G-code* of the Anycubic Orca profile (`; AFTER_LAYER_CHANGE [layer_num] @ [layer_z]mm`) and is a second source for the total.

**Mainsail counter (Rinkhals).** The Orca files contain no `SET_PRINT_STATS_INFO`, so the firmware's own `print_stats.info` stays at 0/0. On your printer, however, Moonraker runs the Rinkhals plugin `kobra.py` (public source, function `patch_status`), which **rewrites every status update on its way to Mainsail**: whenever an update contains `virtual_sdcard.current_layer`, it copies that value into `print_stats.info.current_layer`, and it copies the last `virtual_sdcard.total_layer` it has seen into `info.total_layer`. So Mainsail shows the **same numbers as the touchscreen**, without any `SET_PRINT_STATS_INFO`. That explains the Cube observation: right after the job `info` was still `0/0`, and later it read `2/2` (nothing in the firmware changed it; the plugin fills it when an update carrying the layer values arrives, and a plain query returns Moonraker's cached copy). Nothing in the Moonraker command history shows `SET_PRINT_STATS_INFO`.

Consequences:

- With Rinkhals, **do not add `SET_PRINT_STATS_INFO` to Orca**: the plugin overwrites its values as soon as the next layer update arrives, so it brings nothing (and without the plugin the touchscreen would not need it either).
- If Mainsail still shows wrong layers, the likely causes are in the plugin's bookkeeping, not in the G-code: it keeps the last `total_layer` it received, and resets it to 0 only when a print is started *through Moonraker/Mainsail*; a print started from the touchscreen after another job can keep showing the previous job's total until a new `total_layer` value is sent (inferred from the plugin code, not observed), and the cached `info` is only refreshed when a layer update arrives. What exactly Mainsail shows wrong (before the print, at start, off by one, total) would tell which one it is.
- Orca-side, nothing to change for the layers: keep the Anycubic profile's *Layer change G-code* `; AFTER_LAYER_CHANGE [layer_num] @ [layer_z]mm` (a second source for the total) and keep Orca's own `;LAYER_CHANGE` comment (drives the current layer and the timelapse trigger). The header line `; total layer number: N` is not read by the firmware; the footer line `; total layers count = N` that Orca writes is.

**On a printer without Rinkhals** (stock Anycubic firmware and Moonraker), a web interface only gets `print_stats.info` from `SET_PRINT_STATS_INFO`, so you would add `SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]` to the *Machine start G-code* and `SET_PRINT_STATS_INFO CURRENT_LAYER={layer_num + 1}` to the *Layer change G-code* (total first, because a new total resets the current layer to 0; Orca's `layer_num` starts at 0). Not tested (to verify).
- Nothing found in the firmware clears the layer values at the end of a print; send `TOTAL_LAYER=0` or a new `TOTAL_LAYER` at the next job's start (to verify).
- The command is only informative. Wrong values change nothing in the motion, the temperatures or the file position.

---

*End of chapter 23.*

---

# Chapter 24 — Probe: `PROBE`, `PROBE_ACCURACY`, `QUERY_PROBE`, `PROBE_CALIBRATE`, manual probing (`TESTZ`, `ACCEPT`, `NEXT`, `ABORT`), `Z_OFFSET_APPLY_PROBE`, factory tests `G9100`–`G9104`

The *probe* is the sensor that finds the bed surface. On the Kobra 3 Max it is the strain sensor of the nozzle (the same one `LEVIQ_AUTO_ZOFFSET` uses, chapters 8 and 14; the `[probe]` section of `printer.cfg` is on pin `PB15`, no X/Y offset: the probe point **is** the nozzle position). Almost all of this chapter is the standard Klipper probe code with a few Anycubic changes; the differences are called out.

Stock `[probe]` values (`printer.cfg`): `speed` 6 mm/s, `lift_speed` 12 mm/s, `samples` 2, `sample_retract_dist` 2 mm, `samples_result` `weighted`, `samples_tolerance` 0.04 mm, `samples_tolerance_retries` 20, `x_offset`/`y_offset` 0. The `z_offset` is **not** in the stock file (it is commented out); the live value is stored by `SAVE_CONFIG` in the printer's own configuration file (`printer_mutable.cfg`).

| Command | Purpose |
|---|---|
| `PROBE` | Lower the nozzle at the current X/Y until it touches the bed; report the Z |
| `PROBE_ACCURACY` | Repeat the probing at the current X/Y and print statistics |
| `QUERY_PROBE` | Report whether the probe is currently triggered |
| `PROBE_CALIBRATE` | Calibrate the probe's `z_offset` by hand (paper test) |
| `TESTZ`, `ACCEPT`, `NEXT`, `ABORT` | The manual-probing helper commands (only exist during a manual probe) |
| `Z_OFFSET_APPLY_PROBE` | Turn the current G-code Z offset into the saved probe `z_offset` |
| `G9100`–`G9104` | Factory tests built on `PROBE_ACCURACY` (not for normal use) |

**A rule that holds for every command below (and corrects earlier chapters).** Looking at how the firmware reads numeric parameters: when a value is outside the allowed range (for example `SAMPLES=0` or a negative `PROBE_SPEED`), the firmware **does not stop with an error**. It writes a line `Error on '<command>': <PARAM> must have minimum of …` into its log (not on the console) and **carries on**. For decimal parameters the value used is **the limit itself** (the minimum, maximum, or the bound for "above"/"below"); for integer parameters the value you typed is used unchanged. So a wrong value silently does something else than you typed. Earlier chapters that say "error" for an out-of-range value (for example `M220 S0` in chapter 19) should be read as "logged, and the limit is used" (to verify on the printer).

---

## `PROBE` — Measure the bed height at the current position

### What it does

1. Requires **Z to be homed** (otherwise the command stops with the console message `Must home before probe`).
2. Takes `SAMPLES` measurements in a row. Each one: lowers the nozzle at `PROBE_SPEED` until the sensor triggers, then prints `probe at X,Y is z=…` (the Z the nozzle reached). Between two measurements the nozzle lifts by `SAMPLE_RETRACT_DIST` at `LIFT_SPEED`.
3. After each measurement the spread (highest − lowest of the samples so far) is checked against `SAMPLES_TOLERANCE`. If it is larger, the firmware writes `Probe samples exceed tolerance. Retrying …` **in its log only** (no console message) and starts the sample series again. After `SAMPLES_TOLERANCE_RETRIES` such restarts, the next failure stops the command with the console error `Probe samples exceed samples_tolerance` (the nozzle has then already been lowered several times).
4. Computes **one** result from the samples according to `SAMPLES_RESULT` (see below), lifts the nozzle and prints `Result is z=…`.

The result is also remembered as the probe's "last result" (used internally; no status field was found).

### Syntax

```
PROBE [PROBE_SPEED=<mm/s>] [LIFT_SPEED=<mm/s>] [SAMPLES=<n>] [SAMPLE_RETRACT_DIST=<mm>]
      [SAMPLES_TOLERANCE=<mm>] [SAMPLES_TOLERANCE_RETRIES=<n>] [SAMPLES_RESULT=<mode>]
```

### Parameters

| Parameter | Meaning | Values | Default (stock) |
|---|---|---|---|
| `PROBE_SPEED` | Lowering speed during measurement | mm/s, above 0 | 6 |
| `LIFT_SPEED` | Speed of the lift between samples and at the end | mm/s, above 0 | 12 |
| `SAMPLES` | Number of measurements averaged | integer ≥ 1 | 2 |
| `SAMPLE_RETRACT_DIST` | Lift between two measurements | mm, above 0 | 2 |
| `SAMPLES_TOLERANCE` | Largest allowed spread between samples | mm, ≥ 0 | 0.04 |
| `SAMPLES_TOLERANCE_RETRIES` | How many times the series may be restarted | integer ≥ 0 | 20 |
| `SAMPLES_RESULT` | How the samples are combined | `median`, `weighted`; any other word = plain average | `weighted` |

**`weighted`** (an Anycubic addition): when there are **exactly two samples** (the stock `SAMPLES=2`), the result is `(2 × first sample + 3 × second sample) / 5` (weights 0.4 and 0.6), for X, Y and Z, so the second, more settled measurement counts more. With **any other number of samples** (`SAMPLES=1`, 3, 5…) `weighted` silently becomes the plain average of all samples (verified in the code). `median` is the middle sample; any other value is the average of all samples.

### Examples

```
G28
PROBE
```
One probing with the stock settings (2 samples, weighted result).

```
PROBE SAMPLES=5 SAMPLES_RESULT=median PROBE_SPEED=3
```
Five slower measurements, keep the median.

### Notes and cautions

- **The nozzle goes down until the sensor triggers.** Do it with a clean, cold or at most warm nozzle: hot filament on the nozzle makes the Z result wrong, and the LeviQ3 routines wipe the nozzle first for this reason (chapters 2 and 14).
- Keep the sensor mechanism in mind: the firmware watches the strain sensor between moves; a bed with something on it, or a nozzle with a blob, gives a result that is too high.
- `PROBE` does not change any offset or mesh by itself.

---

## `PROBE_ACCURACY` — Repeatability test at the current position

### What it does

Measures the same spot many times to see how repeatable the probe is.

1. Prints a line `PROBE_ACCURACY at X:… Y:… Z:… (samples=… retract=… speed=… lift_speed=…)`.
2. Repeats `SAMPLES` times: probe down (prints `probe at X,Y is z=…`), lift by `SAMPLE_RETRACT_DIST`.
3. Prints one line with the **maximum, minimum, range, average, median and standard deviation** of the measured Z values.

Unlike `PROBE`, the tolerance check and `SAMPLES_RESULT` are **not** used: every sample is kept.

### Syntax

```
PROBE_ACCURACY [PROBE_SPEED=<mm/s>] [LIFT_SPEED=<mm/s>] [SAMPLES=<n>] [SAMPLE_RETRACT_DIST=<mm>]
```

### Parameters

Same meaning as for `PROBE`. `SAMPLES` defaults to **10** here.

### Examples

```
G28
G1 X210 Y210 F6000
PROBE_ACCURACY SAMPLES=10
```
A range of a few thousandths of a millimetre is normal; a range above about 0.02 mm points to a dirty nozzle, a loose sensor or a loose bed.

### Notes and cautions

- Needs Z homed. Do not leave the printer unattended: the nozzle touches the bed ten times.
- The factory tests `G9100`–`G9103` below are made of `PROBE_ACCURACY` runs.

---

## `QUERY_PROBE` — Is the probe triggered?

### What it does

Reads the state of the probe sensor, up to `COUNT` times (each reading is taken after the last queued move has finished, so it is current):

- a reading that is **not triggered** prints `probe: open` and the next reading follows;
- as soon as a reading is **triggered**, it prints `probe: TRIGGERED` and the command **stops** (the remaining readings are not taken).

So with nothing touching the nozzle you get `COUNT` lines `probe: open`; `COUNT=0` (or negative) prints nothing. (Words and loop verified in the code; not run on the printer.)

### Syntax

```
QUERY_PROBE [COUNT=<n>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `COUNT` | Number of readings | integer | 5 |

### Notes and cautions

- The state of a strain sensor depends on its zero point; a "TRIGGERED" with nothing touching it means the sensor is out of range (see `QUERY_ADC`, chapter 25 / `CS1237_DUMP`, chapter 8).
- The command is harmless: no movement.

---

## `PROBE_CALIBRATE` and manual probing — Set the probe `z_offset` by hand

### What it does

`PROBE_CALIBRATE` finds the offset between "where the sensor triggers" and "where the nozzle really touches the paper", the standard Klipper paper test:

1. Refuses to start if a manual probe is already running.
2. Runs a normal probe (`Run_probe`: same parameters as `PROBE`, so `SAMPLES`, `PROBE_SPEED`… apply) at the current position and remembers the Z.
3. Lifts 5 mm at `LIFT_SPEED`, then moves to the probe point (the X/Y offsets are 0 here, so it stays where it is) at the probe speed.
4. Starts the **manual probe** and prints `Starting manual Z probe. Use TESTZ to adjust position.` and `Finish with ACCEPT or ABORT command.`

You then move the nozzle with `TESTZ` until a sheet of paper just drags, and finish with `ACCEPT`. The new `z_offset` is the Z found by the probe minus the Z you accepted. It is **staged** (3 decimals) in the configuration (`SAVE_CONFIG` is needed to keep it, chapter 17); the result line is written **only to the firmware log**, not to the console (the standard Klipper console message with the value and the "SAVE_CONFIG will…" text is not shown).

### Manual-probe helper commands

These exist **only while a manual probe is running** (they are registered when it starts and removed when it ends):

| Command | Effect |
|---|---|
| `TESTZ Z=<value>` | Move the nozzle in Z. `Z=-1` / `Z=+0.1` = relative move (mm); `Z=--` and `Z=++` = "large" step down/up, `Z=-` and `Z=+` = half the previous step (the bisection used in Klipper); a plain number is an **absolute** Z. Prints `Z position: <lower> --> <current> <-- <upper>`. If the move is too small for the motor: `WARNING: No change in position (reached stepper resolution)`. |
| `ACCEPT` (alias `NEXT`) | Take the current Z as the result and finish. Fails with `Manual probe failed! Use TESTZ commands to position the nozzle prior to running ACCEPT.` (and ends the probe) if X/Y changed or the nozzle is not lower than where the probe started. |
| `ABORT` | End the manual probe without result. |

### Syntax

```
PROBE_CALIBRATE [SPEED=<mm/s>] [PROBE_SPEED=…] [LIFT_SPEED=…] [SAMPLES=…] [SAMPLE_RETRACT_DIST=…]
                [SAMPLES_TOLERANCE=…] [SAMPLES_TOLERANCE_RETRIES=…] [SAMPLES_RESULT=…]
TESTZ Z=<value>
ACCEPT
NEXT
ABORT
```

### Parameters

`PROBE_CALIBRATE`: all the `PROBE` parameters; plus `SPEED` (default 5 mm/s) = speed of the manual moves (a standard Klipper manual-probe parameter, read when the manual probe starts).

### Examples

```
G28
PROBE_CALIBRATE
TESTZ Z=-1
TESTZ Z=-0.1
ACCEPT
SAVE_CONFIG
```

### Notes and cautions

- On this printer the standard procedure is replaced in normal use by `LEVIQ_AUTO_ZOFFSET` (chapter 14), which does the same job with the strain sensor and no paper. Use `PROBE_CALIBRATE` only if you want to calibrate by hand.
- The bed must be at the temperature you print at, and the nozzle clean. `SAVE_CONFIG` does not restart the firmware (chapter 17): the new value is read from the file, but the status of the configuration shown by Moonraker may still show the old one.
- Manual probing is also reachable from `BED_MESH_CALIBRATE METHOD=manual`, and from the touchscreen routines.

---

## `Z_OFFSET_APPLY_PROBE` — Make the current Z offset permanent

### What it does

1. Reads the current **G-code Z offset** (the "babystepping" set with `SET_GCODE_OFFSET Z=` / `Z_ADJUST=`, chapter 19).
2. If it is 0: writes `Nothing to do: Z Offset is 0` and stops.
3. Otherwise computes **new `z_offset` = current probe `z_offset` − offset** (rounded to 4 decimals), and **stages** it in the configuration (`SAVE_CONFIG` writes it, chapter 17). The usual Klipper console message (value and "SAVE_CONFIG will…") is **not** shown; the value goes to the log.
4. Sends two internal events (`leviq2:save_calibration`, `leviq:sync_zoffset`) so that the touchscreen and the LeviQ3 module update their copy of the Z offset.

### Syntax

```
Z_OFFSET_APPLY_PROBE
```

### Parameters

None.

### Examples

```
SET_GCODE_OFFSET Z_ADJUST=0.05 MOVE=1
Z_OFFSET_APPLY_PROBE
SAVE_CONFIG
```
You adjusted the first layer by 0.05 mm while printing; this stores the correction permanently.

### Notes and cautions

- Nothing is written to the file until `SAVE_CONFIG`.
- Besides the G-code command, the touchscreen reaches the same function through the web interface commands `move/z_offset_apply` and `move/z_offset_apply_absolute` (the second takes an absolute offset). The touchscreen also uses an internal "move Z" request that moves the nozzle, waits 2 s and logs the strain-sensor reading.
- `LEVIQ_AUTO_ZOFFSET` calls this routine itself (chapter 14).

---

## `G9100`–`G9104` — Factory probe tests

Five commands used by Anycubic's production tests. They are **not for normal use**: they move the head, heat, run many probings and (G9104) rewrite calibration values. All parameters are integers and default to **0**, so they must be given explicitly. They were read from the code only and never tried; the parameter roles and the fixed temperatures/positions above were checked in the disassembly, the measurement steps of `G9104` only at call-summary level (to verify).

| Command | What it does |
|---|---|
| `G9100 S<samples> X<x> Y<y> F<speed>` | Homes `G28` if X/Y/Z are not all homed, waits 3 s, moves to (X,Y) at feed rate F (mm/min), then runs `PROBE_ACCURACY SAMPLES=S`. Test of the Z-axis zeroing repeatability at one point. |
| `G9101 S<rounds> C<samples>` | `G90`, then `S` times: `G9111` (heating and wiping preparation with its default values, chapter 2) followed by `PROBE_ACCURACY SAMPLES=C`; at the end `TURN_OFF_HEATERS` and `M107`. Test of the wiping (a dirty nozzle gives a worse accuracy). |
| `G9102 S<rounds> C<samples> F<speed>` | Homes if needed (`G28`, when X/Y/Z are not all homed), `G90`, waits 3 s, then repeats `S` times: for each of five bed points — (15,15), (240,15), (125,125), (15,240), (240,240) — move there at F mm/min, wait 1 s, `PROBE_ACCURACY SAMPLES=C`, lift Z by 10 mm (`G91 G1 Z10 F300`). Test of the bed shake / bed stability. |
| `G9103 S<rounds> C<samples> F<speed>` | Fixed preparation: `HOME_XY`, bed 60 °C and nozzle 170 °C (`M140 S60`, `M109 S170`), `G9112` (wiping), then bed 60 °C and nozzle 140 °C, `M107`, `G28`, `G90`, wait 3 s. Then, `S` times: `M204 S10000`, `G1 F<F>`, move (with a short wait) to the first three positions of `bed_mesh_error_pos` in `printer.cfg` — (10,10), (410,10), (205,205) — running `PROBE_ACCURACY SAMPLES=C` at them, logging `point_diff_<n> val1:… val2:… diff:…` (the difference between two measured points) in the firmware log, and lifting Z by 10 mm. Finishes with `G1 X0 Y0 F6000`, `TURN_OFF_HEATERS`, `M204 S20000`, `M84`. Test of the bed-mesh height error. |
| `G9104 [R]` | Calibration of the Z gap with the strain sensor, for **one side per run**: with the parameter `R` (any value) the **right** side (`G28 R`, position `calibration_right_position` of `[probe]`), without it the **left** side (`G28 L`, `calibration_left_position`). Steps: `G90`, `BED_MESH_CLEAR`, `G9121 E1 R1 T0.1` (strain sensor mode, chapter 8), `G28 R` / `G28 L` (chapter 12), wait 2 s, read the zero ADC value, then Z moves in steps while the ADC readings are logged; the two measured gaps are stored as `z_gap_right_up` / `z_gap_right_down` (R) or `z_gap_left_up` / `z_gap_left_down` (no R) in the configuration and saved (`save_config`); ends with `M84` and `G9121 E0 R0`. Changes saved calibration values: do **not** run it. |

### Notes and cautions

- These are the commands behind the factory "test" screens; running one by accident (even `G9100` with all defaults, which moves to X0 Y0 at F0) is possible but pointless.
- `G9101`–`G9103` heat, wipe and probe for many minutes unattended; `G9104` writes the probe-gap calibration.

---

## Link with the LeviQ3 commands (chapter 14)

The LeviQ3 auto-leveling commands are built on the probe described in this chapter. Both end up measuring with the same strain sensor.

| LeviQ3 command | What it uses from this chapter |
|---|---|
| `LEVIQ_PROBE` | Runs `BED_MESH_CALIBRATE ADAPTIVE=1` (chapter 3); every mesh point is a probing like `PROBE`. The `[probe]` settings apply: speed 6 mm/s, 2 samples, `weighted` result, tolerance 0.04 mm, 20 retries. |
| `LEVIQ_AUTO_ZOFFSET` | Does **not** use `PROBE`: it steps Z and reads the strain sensor itself. It ends by calling `Z_OFFSET_APPLY_PROBE`, which stores the result as the probe `z_offset`. |
| `LEVIQ_TEMP_OFFSET`, `LEVIQ_SET_ZERO` | Adjust the probe `z_offset` and the G-code Z offset, the same mechanism as `Z_OFFSET_APPLY_PROBE`. |
| `LEVIQ`, `LEVIQ_PREHEATING`, `LEVIQ_WIPING` | No direct link: they only prepare the nozzle and bed (heating, wiping) so that probing is accurate. |

Where the two meet:

- **Failed sample series.** `LEVIQ_PROBE` catches the console error `Probe samples exceed samples_tolerance` (the retries of `PROBE` are used up) and a cancel request. For these it returns its own error instead of the raw message, and for any other failure a generic error; in all cases it switches the strain-sensor chip off. A dirty nozzle that makes the samples disagree therefore ends the leveling with a probe error rather than a half-finished mesh. The exact error codes behind it were not read (to verify).
- **Offset events.** `Z_OFFSET_APPLY_PROBE` sends the event `leviq:sync_zoffset`; the LeviQ3 module listens to it and updates its own copy of the offset.
- **Duplicate probing code.** The LeviQ3 module contains its own copy of the probing routine: it reads the same parameter names as `PROBE`, uses the same `weighted` / `median` calculation, and adds a `G90`, a wait and a dwell between samples. Whether `BED_MESH_CALIBRATE ADAPTIVE=1` runs the standard routine or this copy was **not confirmed** (the link is a method reference whose caller was not found); either way the `[probe]` settings apply (to verify).

**Practical consequence:** the probe settings used by `LEVIQ_PROBE` are the ones a bare `PROBE` uses, so `PROBE_ACCURACY` at a spot of the bed shows how reliable the leveling will be there.

---

*End of chapter 24.*

---

# Chapter 25 — Queries

Three read-only commands that report the state of the printer's electronics: the end-of-travel (endstop) switches and the analog inputs (ADC). They move nothing and change no setting.

**Important: in this firmware version none of the three works as intended** (found by reading the code, not yet confirmed on the printer; see "To verify" at the end of the chapter). Do not rely on them for diagnosis.

| Command | What it should do | What it does in this firmware |
|---|---|---|
| `M119` | Report the state of every endstop | Same command as `QUERY_ENDSTOPS`: ends with a console error as soon as at least one endstop is registered (to verify) |
| `QUERY_ENDSTOPS` | Report the state of every endstop | Same as above |
| `QUERY_ADC NAME=<adc>` | Report the last value of an analog input | Never gives a value: always answers with the list of ADC names, preceded by empty entries |

---

## `QUERY_ENDSTOPS` / `M119` — Endstop state

`M119` and `QUERY_ENDSTOPS` are the same command (`M119` is the usual name used by slicers and host programs, and has no help text).

### What it should do

1. Wait until the planned moves are finished being prepared (the move queue is flushed up to the last planned move).
2. Ask each endstop of the printer for its state at that moment.
3. Print one line, `name:state` per endstop separated by spaces, for example `x:open y:open z:open`. `open` = not pressed / not triggered, `TRIGGERED` = pressed / triggered.

### What it does in this firmware

Step 1 runs, then the command stops with an internal type error ("interface conversion", shown in the console) on the **first** endstop, so no line is printed. It is a plain console error: the printer does **not** go into shutdown and nothing else is affected. Why: the code reading the list of endstops expects a different kind of entry than the one that was stored. It is the same fault as in the public source code of this program (which this firmware never fixed).

The printer registers its endstops through the rails of the three axes (stock names `x`, `y`, `z`), so on a stock configuration `M119` is affected. If a configuration had no endstop at all, the command would print an empty line.

### Syntax

```
M119
QUERY_ENDSTOPS
```

### Parameters

None.

### Examples

```
M119
```
Expected on a normal Klipper machine: `x:open y:open z:open`. On this firmware: console error, no result.

### Notes and cautions

- **The three endstops are not mechanical switches.** In the stock configuration X and Y use the stepper drivers' "virtual endstop" (a motor stall is detected) and Z uses the probe (the strain sensor under the nozzle: `probe:z_virtual_endstop`). So even a working `M119` would show the driver/probe state, not a switch you can press. `TRIGGERED` outside homing is not something you can interpret like a switch.
- **The status read by the web interface** (the Moonraker/Mainsail request `query_endstops/status`, and the `query_endstops` object) is also unreliable: it uses the same list, and the code that turns the state into `open` / `TRIGGERED` compares the endstop *name* instead of its state, so every endstop comes out as `TRIGGERED`. A test on the idle printer (2026-11-13) with this web request returned `{"x": "TRIGGERED", "y": "TRIGGERED", "z": "TRIGGERED"}`, which fits this explanation but is also what a really triggered state would look like (to verify: the three values are not proof).
- To check the Z/probe sensor, `QUERY_PROBE` (chapter 24) is the working command: it reads the probe state and reports `open` / `TRIGGERED` for the probe.
- Querying the state calls the flush of the move queue, so with moves still queued the command waits for them. It never moves the head.

---

## `QUERY_ADC` — Last value of an analog input

### What it should do

1. Take the name of an analog input registered by the printer (typically the temperature sensors: nozzle thermistor, bed thermistor, MCU temperature, …).
2. Print `ADC object "<name>" has value <v> (timestamp <t>)`: `v` is the last reading as a fraction of full scale (0.000000 to 1.000000) and `t` the time of that reading in seconds.
3. With `PULLUP=<ohms>`, add a second line ` resistance <r> (with <pullup> pullup)` with the resistance of the sensor that would give this reading.
4. With an unknown name, list the available names.

### What it does in this firmware

1. `NAME` is read (missing NAME → error `Error on 'QUERY_ADC': missing NAME` in the console).
2. The code tests the name the wrong way round and decides that the name is *never* known. Whatever you type — an existing name, a wrong one, nothing meaningful — it ends with step 4.
3. It prints `Available ADC objects: ` followed by the names, sorted alphabetically, separated by `, `. Because of a second mistake in the list building, the names are **preceded by as many empty entries as there are ADC objects** (two ADCs would give `Available ADC objects: , , heater_bed, extruder`-like output with two leading `, `).
4. No value is ever read and `PULLUP` is never evaluated.

So the only use of the command in this firmware is to see which ADC names exist (after skipping the commas).

### Syntax

```
QUERY_ADC NAME=<adc name> [PULLUP=<ohms>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `NAME` | Name of the analog input | A name registered by the printer | none — required |
| `PULLUP` | Pull-up resistor of the sensor circuit, to compute a resistance | Number (ohms); 0 = no resistance | 0 |

### Examples

```
QUERY_ADC NAME=list
```
Not an ADC name, so it prints the list of names (this is also what any valid name does in this firmware).

### Notes and cautions

- **If the value reading worked**, `PULLUP` would still give a wrong resistance: the formula limits the reading with the wrong bound, so the result would be `PULLUP × 99999` for any reading below 1.0 (and a division by zero at exactly 1.0). The usual resistance formula is `PULLUP × v / (1 − v)`. Do not trust a resistance computed by this command.
- For temperatures, use `M105` (chapter 11), which works and gives the converted values.
- Which ADC names are registered was not determined (the registration is done through an indirect call that was not traced). Sensor names in the configuration (`extruder`, `heater_bed`, …) are the usual ones in Klipper.

---

### To verify (not tested on a printer)

- `M119` ending with a console error (and not printing): the fault was read in the disassembly (stored entry = pointer to a list, read back as a list value), and the idle printer answered to the web request but this was a different code path. A single `M119` on the idle printer would confirm it (it is read-only).
- `QUERY_ADC NAME=extruder` (or any name) printing only `Available ADC objects: …`; if it prints a value instead, the analysis of the test is wrong.
- The names registered as ADC objects.

---

*End of chapter 25.*


---

# Chapter 26 — TMC drivers

The motors of the printer are driven by **TMC2209** stepper drivers, controlled over a serial (UART) link. This chapter covers the six commands that act on those drivers directly: change the motor current, switch the quiet "StealthChop" mode on or off, write a single driver setting, re-send all settings, and dump the driver registers.

Stock configuration (`printer.cfg`): one driver per motor, named after the motor section:

| `STEPPER=` value | Motor | Run current | Hold current |
|---|---|---|---|
| `stepper_x` | X axis | 1.5 A | 1.1 A |
| `stepper_y` | Y axis (left) | 2.0 A | 1.4 A |
| `stepper_y1` | Y axis (second motor, Max only) | 2.0 A | 1.4 A |
| `stepper_z` | Z axis | 2.0 A | 1.5 A |
| `extruder` | Extruder | 0.8 A | 0.4 A |

All six commands are *multiplexed* commands: they need `STEPPER=<name>` from the table. Without it the console shows `Error on '<command>': missing STEPPER`; with an unknown name it shows `The value … is not valid for STEPPER` (generic wording of the firmware's multiplexed-command handler, as in chapter 13). Neither case stops the printer.

| Command | What it does |
|---|---|
| `SET_TMC_CURRENT` | Change the run and/or hold current of a motor (until restart) |
| `OPEN_STEALTHCHOP` | Put the driver into quiet StealthChop mode |
| `CLOSE_STEALTHCHOP` | Put the driver into the stronger, louder SpreadCycle mode |
| `SET_TMC_FIELD` | Write one named setting of the driver |
| `INIT_TMC` | Re-send all remembered settings to the driver |
| `DUMP_TMC` | Print all the registers of the driver |

**Caution for the whole chapter:** these commands change how the motors are powered. A wrong value can make a motor lose steps, overheat or stop being detected during homing (X and Y homing relies on the driver's stall detection, chapter 12). Nothing here is saved to the configuration: after a restart or power-cycle the values from `printer.cfg` return. Do not use them during a print unless you know what you are doing.

---

## `SET_TMC_CURRENT` — Motor current

### What it does

1. Reads `CURRENT` (run current) and `HOLDCURRENT` (hold current), both optional.
2. If neither is given, nothing is changed.
3. If at least one is given: the missing one is replaced by the *current* run value, respectively by the *requested* hold value (the one from the configuration or from the last command). The driver is updated at the end of the moves already planned (the move queue is flushed first, so a running print is not interrupted but the change comes after the queued moves).
4. The driver stores the current as a 5-bit step (32 levels), so the value actually applied is rounded to the nearest step, and the driver's sensitivity range (the "vsense" bit) is switched when needed for high currents.
5. **No message is shown on the console.** The resulting values (`Run Current: x.xxA Hold Current: y.yyA`) are only written to the printer's log file. (The standard Klipper prints them; this firmware does not.)

### Syntax

```
SET_TMC_CURRENT STEPPER=<name> [CURRENT=<amps>] [HOLDCURRENT=<amps>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Motor / driver (see table above) | `stepper_x`, `stepper_y`, `stepper_y1`, `stepper_z`, `extruder` | none — required |
| `CURRENT` | Run current, applied while the motor moves | 0 to 2.4 A | unchanged |
| `HOLDCURRENT` | Current while the motor stands still | 0 to 2.4 A | unchanged (the hold current is also limited to the run current when applied) |

The upper limit of 2.4 A is the limit of this firmware's driver code (the standard Klipper limit for this driver is 2.0 A). Values outside 0–2.4 are **not refused**: the firmware only writes a line to its log and uses the nearest limit (general behaviour of the number parameters, see the audit note in chapter 24).

### Examples

```
SET_TMC_CURRENT STEPPER=stepper_z CURRENT=1.8
```
Lowers the Z run current to about 1.8 A (rounded to the nearest driver step); the hold current is not changed.

```
SET_TMC_CURRENT STEPPER=extruder CURRENT=0.8 HOLDCURRENT=0.3
```
Sets both currents of the extruder motor.

### Notes and cautions

- Nothing is printed, so to check what the driver really uses, run `DUMP_TMC` and read the `IHOLD_IRUN` and `CHOPCONF` lines (the `irun`, `ihold` and `vsense` values), or look at the `run_current` / `hold_current` values in Moonraker (`tmc2209 <name>` object).
- The change lasts until a restart (it is not written by `SAVE_CONFIG`).
- A higher current gives more torque but heats the motor and driver; the stock values were chosen by the manufacturer.

---

## `CLOSE_STEALTHCHOP` — SpreadCycle mode

### What it does

Switches the driver of one motor from the quiet **StealthChop** mode to **SpreadCycle** (the normal chopper mode: more torque and more precise at high speed, but noisier). Steps:

1. Sets the driver setting that selects SpreadCycle (`en_spreadcycle` = 1 on the TMC2209).
2. Writes the `GCONF` register to the driver. A line with the old and new values goes to the log only.
3. Sets the stall-detection speed threshold `TCOOLTHRS` to 0 and writes it to the driver (this switches off the stall-detection ("StallGuard") range that sensorless homing uses; it is the normal state outside homing).

The commands act immediately (they are not synchronised with the move queue).

The stock configuration has `stealthchop_threshold: 0` on all drivers, which means **StealthChop is the normal mode of the printer** and `CLOSE_STEALTHCHOP` is the exception. The firmware itself runs `CLOSE_STEALTHCHOP` on X and Y (and Y1 on the Max) when a print starts (`M117`, chapter 5) and after the homing used by the power-loss recovery (chapter 22), and `OPEN_STEALTHCHOP` at the end/cancel of a print (chapters 21 and 5).

### Syntax

```
CLOSE_STEALTHCHOP STEPPER=<name>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Motor / driver | see the table at the start of the chapter | none — required |

### Examples

```
CLOSE_STEALTHCHOP STEPPER=stepper_x
CLOSE_STEALTHCHOP STEPPER=stepper_y
```
X and Y motors in SpreadCycle, the way the printer runs while printing.

### Notes and cautions

- There is no message. The effect can be read with `DUMP_TMC` (`GCONF` line, field `en_spreadcycle`, and the `TCOOLTHRS` line).
- The next homing (`G28`) temporarily sets StealthChop and the stall threshold itself, and puts back what it found afterwards, so the mode you set is restored after homing.
- For drivers that have an `en_pwm_mode` setting instead of `en_spreadcycle` (not the case on this printer), the code writes 1 into `en_pwm_mode`, which would mean the opposite (to verify, irrelevant for the TMC2209 of the printer).

---

## `OPEN_STEALTHCHOP` — StealthChop mode

### What it does

The reverse of `CLOSE_STEALTHCHOP`:

1. Remembers the current `TCOOLTHRS` value.
2. Sets the driver setting that selects StealthChop (`en_spreadcycle` = 0).
3. Writes the `GCONF` register (the change is logged to the log file only).
4. **Only if `TCOOLTHRS` was 0**: sets it to its maximum 1 048 575 (hex `FFFFF`) and writes the `TCOOLTHRS` register. If it already had another value, it stays.

Setting `TCOOLTHRS` to the maximum is what the firmware also does during sensorless homing (stall detection active at all speeds). So after `OPEN_STEALTHCHOP` the driver is left with the stall detection enabled; the effect of this on normal printing moves was not analysed (to verify).

### Syntax

```
OPEN_STEALTHCHOP STEPPER=<name>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Motor / driver | see the table at the start of the chapter | none — required |

### Examples

```
OPEN_STEALTHCHOP STEPPER=stepper_x
```
X motor back to the quiet mode.

### Notes and cautions

- The firmware itself uses it at the end of a print and when it cancels, so the printer is in the quiet mode when idle.
- A subsequent `G28` temporarily sets the homing settings again and restores `TCOOLTHRS` to 0 afterwards (a known quirk of the homing code: it always writes 0, whatever the earlier value was).
- Like `CLOSE_STEALTHCHOP`, there is no message.

---

## `SET_TMC_FIELD` — Write one driver setting

### What it does

1. Reads `FIELD` and converts it to lower case.
2. If the name is not a known driver setting: the command ends with the error `Unknown field name '<name>'`. **This error is returned by the command (not just displayed), so the printer goes into shutdown** (it must be power-cycled, not `FIRMWARE_RESTART`, see chapter 5). Check the name carefully.
3. Otherwise the new `VALUE` is put into that setting (only that bit-field; other settings of the same register are kept) and the whole register is written to the driver at the end of the planned moves.
4. No message is shown.

### Syntax

```
SET_TMC_FIELD STEPPER=<name> FIELD=<field> [VALUE=<number>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Motor / driver | see the table at the start of the chapter | none — required |
| `FIELD` | Name of the driver setting (case does not matter) | any TMC2209 field, e.g. `sgthrs`, `tcoolthrs`, `tpwmthrs`, `en_spreadcycle`, `toff`, `hstrt`, `hend`, `tbl`, `mres`, `intpol`, `irun`, `ihold`, `iholddelay`, `pwm_autoscale`, `semin`, `semax`; the names are those of the TMC2209 data sheet | none — required |
| `VALUE` | New value of the setting | integer, written as is | **0** (VALUE is not required: if you forget it, the setting is set to 0!) |

### Examples

```
SET_TMC_FIELD STEPPER=stepper_x FIELD=sgthrs VALUE=70
```
Changes the stall sensitivity of the X motor (stock 73) used by the sensorless homing; higher = more sensitive. Lasts until restart.

```
SET_TMC_FIELD STEPPER=stepper_y FIELD=en_spreadcycle VALUE=1
```
Same effect on the mode as `CLOSE_STEALTHCHOP` for the mode setting only.

### Notes and cautions

- A forgotten `VALUE` writes 0: `SET_TMC_FIELD STEPPER=stepper_x FIELD=irun` would set the run current step to 0 (motor without torque).
- Whether an out-of-range value is cut to the size of the field was not examined in the firmware (standard Klipper code cuts it to the field width). Use values that fit the field.
- Prefer `SET_TMC_CURRENT` for the currents.
- The setting is lost at restart.

---

## `INIT_TMC` — Re-send the driver settings

### What it does

1. Writes a line `INIT_TMC <name>` in the log.
2. Waits for the planned moves (the move queue is flushed).
3. Sends **every register value the firmware remembers** for this driver to the driver again (for example the current, mode, stall threshold, microstepping).

This is the same action the firmware does once at start-up and each time the motors are enabled, so the driver gets back its configured state (use it after a driver reset, for example a short power glitch, or after `SET_TMC_FIELD` experiments: it re-sends the *remembered* values, which include your `SET_TMC_FIELD` changes, so it does **not** restore the configuration file values).

### Syntax

```
INIT_TMC STEPPER=<name>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Motor / driver | see the table at the start of the chapter | none — required |

### Examples

```
INIT_TMC STEPPER=stepper_z
```
Re-sends the settings of the Z driver.

### Notes and cautions

- No console message.
- To return to the file values after experiments, restart the printer (power-cycle).

---

## `DUMP_TMC` — Read the driver registers

### What it does

1. Writes `DUMP_TMC <name>` in the log and waits for the planned moves.
2. Prints `========== Write-only registers ==========` and one line for each register the firmware only writes, showing the values it last sent.
3. Prints `========== Queried registers ==========` and then reads each readable register **from the driver** and prints it. The readable registers of the TMC2209 are: `GCONF`, `GSTAT`, `IFCNT`, `OTP_READ`, `IOIN`, `FACTORY_CONF`, `TSTEP`, `MSCNT`, `MSCURACT`, `CHOPCONF`, `DRV_STATUS`, `PWMCONF`, `PWM_SCALE`, `PWM_AUTO`, `SG_RESULT`.
4. Each line has the form `NAME        0000xxxx field=value field=value …`: register name, the 32-bit value in hexadecimal, then the decoded fields.

The write-only registers come out in no fixed order (the firmware goes through its list without sorting).

### Syntax

```
DUMP_TMC STEPPER=<name>
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `STEPPER` | Motor / driver | see the table at the start of the chapter | none — required |

The standard Klipper `REGISTER=` parameter (dump one register) is **not** available in this firmware: the command always dumps everything.

### Examples

```
DUMP_TMC STEPPER=stepper_x
```
Shows all registers of the X driver: use `DRV_STATUS` (over-temperature, short circuits, stand-still), `SG_RESULT` (stall measure) and `IFCNT` (communication check, grows with each write).

### Notes and cautions

- All drivers of the printer share one serial line (`PA3` of the main board; the extruder driver sits on the nozzle board), so a dump takes a short while. It does not move anything.
- If a driver does not answer, the value read is shown as is (or the command reports a communication error in the console — not examined).

---

### To verify (not tested on a printer)

- Everything in this chapter comes from reading the firmware; no command was sent.
- The exact console wording of the multiplexed-command errors (taken from chapter 13).
- Effect on the motors of `OPEN_STEALTHCHOP` (`TCOOLTHRS` = FFFFF left in place) outside homing.
- `SET_TMC_FIELD` on an out-of-range value and on a register field name list of the TMC2209 (the example names come from the standard TMC2209 field list).
- Read-only way to check without risk: `DUMP_TMC STEPPER=stepper_x` on the idle printer.

---

*End of chapter 26.*


---

# Chapter 27 — Timelapse

Two commands drive the timelapse function: `SNAPSHOT` (send one message to the camera program) and `TIMELAPSE` (send a message and/or switch the function on or off). The firmware itself does not handle pictures or video. It only sends short text messages (`0`, `1` or `2`) over the network to the printer's **camera program** (`gkcam`), which grabs the pictures and builds the video. What `gkcam` does with each message is explained in the section "What the camera program does" below (read from its code).

**You only need this chapter if the printer has the Anycubic camera.** Without a camera the messages go nowhere and nothing visible happens (errors in sending are only written to the log).

| Command | What it does |
|---|---|
| `SNAPSHOT V=<0\|1\|2>` | Send one message to the camera program |
| `TIMELAPSE [V=<0\|1\|2>] [E=<0\|1>]` | Send a message (as `SNAPSHOT`) **and/or** switch the timelapse function on or off |

## How the messages work

Each message is the single character below, sent as a network (UDP) packet to port 10086 of the printer itself:

| Message | Meaning for the camera program |
|---|---|
| `0` | Prepare a new timelapse video (sent at the start of a print) |
| `1` | Add one frame to the video now |
| `2` | Sent by the firmware at the end/cancel, but the camera program does **not** act on it (see below) |

If the packet cannot be sent, an error line goes to the log and the command still finishes without an error on the console.

## What the firmware does by itself during a print

When the timelapse function is **on** (flag set by `TIMELAPSE E=1`, or by the touchscreen through the web request `timelapse/set_config`; **off by default after every restart**):

1. **Print start**: sends `0` and marks the timelapse as *running*.
2. **Each new layer** (the firmware detects layer changes from the `;LAYER_CHANGE` / `;LAYER:` lines of the file): if a timelapse is running, takes a picture. How depends on the setting `timelapse_type` that the slicer writes in the G-code header (`; timelapse_type = 0` or `1`; the sample files of the printer all have `0`):
   - **`0`** — the picture is taken where the head is (sends `1`). Any parking of the head must be in the G-code itself.
   - **`1`** — the firmware parks the head for each picture:
     1. Retract and lift: `G91`, `G1 Z0.4 E-<r> F300`, `G90`, where `<r>` is the retraction length given in the file header (`; filament_retract_length = …`), or **1.5 mm** if the header has none (or a value of 0 or less).
     2. Saves the position/state (`SAVE_GCODE_STATE NAME=timelapse`).
     3. Moves to the picture position `G1 X<snap_x_pos> Y<snap_y_pos> F12000` (stock `[timelapse]`: X457 Y220 — the right edge of the bed, middle depth) and waits for the moves to finish (`M400`).
     4. Waits 0.5 s so the camera image is still, sends `1` (picture), then `G4 P400` (0.4 s).
     5. Returns with `RESTORE_GCODE_STATE NAME=timelapse MOVE=1 MOVE_SPEED=12000` (the speed number is in mm/s and is much higher than the printer can do, so the normal speed limits decide how fast the head returns).
     6. Un-retracts: `G91`, `G1 E<r> F300`, `G90`.
   - any other value: no picture.
3. **End of print** (finish): only if running:
   - type `1`: if X and Y are not homed, `G28 X Y`; moves to the picture position (`G1 X… Y… F12000`, `M400`), sends `1` (last picture), waits 0.4 s, sends `2` (no effect in the camera program; it finishes the video itself), and then `M84` (motors off, to verify).
   - type `0`: sends `1`, waits 0.4 s, sends `2`; no movement.
4. **Cancel**: if running, sends `2` and clears the running mark. (The camera program finishes the video by itself when the print state becomes cancelled.)

The commands `SNAPSHOT` and `TIMELAPSE` below send their messages **directly**, whether the function is on or not and whatever the print state is.

## What the camera program does (`gkcam`)

`gkcam` is a separate program started with the printer (it also does the live camera view and the cloud video link). The timelapse part works like this (read from its code, not tested):

1. **It must be armed.** The printer's app service (`gkapi`, for the touchscreen / cloud app) sends `gkcam` a JSON message over the network link between the two programs. `gkcam` reads the address and port of that link from its configuration file (`api.cfg`; not examined further, it holds credentials). The message has this shape (the names are the ones searched for in the code; the exact framing around it was not traced):

   ```
   {"video_stream_request": {"id": <number>, "method": "openDelayCamera",
                             "params": {"filepath": "<path of the .gcode file>"}}}
   ```

   - `openDelayCamera` arms the timelapse and remembers the `filepath`. If the camera is not ready it answers with the error `11401`; if `filepath` is missing it answers with an error without text.
   - `closeDelayCamera` disarms it.
   - The method name is matched by "contains", so extra text around it does not matter.
   - Without arming, frames (`1`) are ignored. The arming is cleared by the end of the print (see 5), so it must be sent again for every print.

   Nothing in G-code or in the firmware's `[timelapse]` section arms it: the `E=1` flag and this arming are two separate switches, and a video only appears if both are on.
2. **Message `0`**: creates the folder `/useremain/app/gk/Time-lapse-Video/`, deletes any leftover `*.tmp` file there, and opens a new video file `timelapse.tmp` (1280 x 720, H.264 in an MP4 container).
3. **Message `1`**: marks that the next camera image must be added; the camera loop takes the next frame and writes it into the video, then clears the mark. So each `1` adds exactly **one frame** (one frame per layer is what makes it a timelapse).
4. **Message `2`**: the code that receives the messages only looks for `0` and `1`; `2` is ignored.
5. **End of the video**: `gkcam` also watches the print status itself (it receives the `print_stats` state of the printer). When the state becomes `complete` or `cancelled` and the timelapse was armed, it disarms, closes the video and renames `timelapse.tmp` into the final file (see "Name of the finished video" below). It also writes a `.jpg` preview image. Before that it keeps the folder to at most 20 entries: while there are more than 19, it deletes the oldest.

### Name of the finished video

The name is **not** a time stamp. It is built like this:

1. Take the `filepath` given when arming; keep only the part after the last `/`, cut at the first `.gcode`. Example: `/useremain/app/gk/gcodes/3DBenchy_18m.gcode` gives `3DBenchy_18m`.
2. Look for the first free number from `01` to `99`: the first `NN` for which `<name>_NN.mp4` does not yet exist in the folder (so a number freed by deleting a video is used again).
3. The file is `/useremain/app/gk/Time-lapse-Video/<name>_<NN>.mp4`, for example `3DBenchy_18m_01.mp4`, then `3DBenchy_18m_02.mp4` for the next print of the same file.
4. The preview image is named `<name>_<NN>_<frames>.jpg` where `<frames>` is the number of frames in the video (counted as `1` messages received since the last `0`); older previews of the same video name are removed first.

If all 99 numbers are taken, the number part stays empty (the name would end in `_.mp4`; inferred from the code, not a normal case).

So the video is finished by the print state, not by the `2` message. A video is also produced on a cancel (with the frames taken so far). Videos are on the printer in `/useremain/app/gk/Time-lapse-Video/`.

---

## `SNAPSHOT` — Send a message to the camera program

### What it does

Reads `V` and sends the corresponding message: `V=0` → `0` (prepare a new video), `V=1` → `1` (add one frame), `V=2` → `2` (ignored by the camera program). Any other value does nothing. No movement, no check of the on/off flag, no answer on the console.

### Syntax

```
SNAPSHOT [V=<0|1|2>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `V` | Message to send | `0` prepare a new video, `1` add one frame, `2` no effect | 0 (so a bare `SNAPSHOT` sends **start**, not a frame!) |

### Examples

```
SNAPSHOT V=1
```
Take one picture where the head is now.

### Notes and cautions

- `SNAPSHOT` alone sends `0`, which restarts the video file (the frames already in it are lost, because `timelapse.tmp` is deleted and recreated); write `V=1` for a frame.
- For a picture at a fixed spot, put the moves yourself before the command (for example `G1 X457 Y220 F12000`, `M400`, `G4 P500`, `SNAPSHOT V=1`).
- Because the printer's own timelapse sends `1` at each layer when it is on, using `SNAPSHOT V=1` at the same time adds extra pictures.

---

## `TIMELAPSE` — Control the timelapse function

### What it does

1. Reads `V` (default −1). If it is 0, 1 or 2, sends that message exactly as `SNAPSHOT` does; otherwise nothing is sent.
2. Reads `E` (default −1). `E=1` switches the timelapse function **on**, `E=0` switches it **off**; other values leave it as it is.

The `E` flag is the one the print-start / layer / end actions above check. It can also be set from the touchscreen (web request `timelapse/set_config` with `enable` = 0 or 1). The flag is **not saved**: after a restart it is off again.

### Syntax

```
TIMELAPSE [V=<0|1|2>] [E=<0|1>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `V` | Message to send to the camera program (same as `SNAPSHOT`) | `0`, `1`, `2` | −1 (nothing sent) |
| `E` | Switch the automatic timelapse on or off for the next print(s) | `1` on, `0` off | −1 (unchanged) |

### Examples

```
TIMELAPSE E=1
```
Turns the automatic timelapse on (a start message is sent at the next print start, then a picture at every layer).

```
TIMELAPSE E=0 V=2
```
Turns it off and sends `2` (which the camera program ignores; the video is finished by the end of the print, see above).

### Notes and cautions

- In a slicer's start G-code, `TIMELAPSE E=1` can enable the function per print; the function has to be on **before** the print starts for the start message to be sent (the start message is sent when the print is started by the firmware, so a command inside the file comes too late for `0`).
- Parameters are written like `V=1` (with `=`); the values are whole numbers.

---

## Using a USB camera, and Rinkhals

**Is the camera special?** Probably not. `gkcam` opens cameras the standard Linux way (`/dev/video0`, `/dev/video1`, …, Video4Linux) and the strings of the program show no brand or USB-ID check, so the Anycubic camera is most likely an ordinary USB (UVC) camera and another one may work (not tested). Two conditions: the camera must be able to deliver the 1280 x 720 size the timelapse video uses, and `gkcam` must be able to open it (it fails with "UsbCameraOpen … failed" otherwise).

**With Rinkhals the camera belongs to `mjpg_streamer`.** The Rinkhals app "mjpg-streamer" (read in the Rinkhals source on GitHub, not checked on the printer) works like this:

- For camera 0, if a resolution is chosen in the Rinkhals app settings, it **stops `gkcam`** and starts `mjpg_streamer` on that camera (HTTP port 8080; the Mainsail webcam address `/webcam/?action=stream` and the snapshot address `/webcam/?action=snapshot` come from it). A camera can normally be opened by one program at a time, so the stock timelapse of this chapter cannot work in this state: `SNAPSHOT` and `TIMELAPSE` send their messages, but nobody is there to use them.
- The setting for camera 0 has the choice **`Anycubic`**: `mjpg_streamer` then leaves the camera alone and `gkcam` keeps it (stopping the app also restarts `gkcam`). The Mainsail live view then stops working.
- A second USB camera is served by `mjpg_streamer` on port 8081 (a third on 8082, …), and does not conflict with `gkcam`, which uses camera 0.
- Rinkhals has **no timelapse function of its own** (the repository has no timelapse code or Moonraker timelapse component; only the live stream and snapshots).

**Checks on the printer (read-only, over SSH):**

```
ls -l /dev/video*
lsusb
cat /sys/class/video4linux/video0/name
ps | grep -E "gkcam|mjpg"
```

The first shows whether the system sees a camera, the second which USB devices are connected, the third the camera name, the fourth which of the two programs is running (if `mjpg_streamer` is running, `gkcam` is not). In the printer of this documentation the camera is used by `mjpg_streamer`, and Moonraker lists one webcam (`mjpegstreamer-adaptive`, `/webcam/?action=stream`).

**Ways to get a timelapse with a USB camera**

| Option | How | Trade-off |
|---|---|---|
| A. Keep `mjpg_streamer`, make your own timelapse | A script (on the printer or another computer) watches the layer number in Moonraker (`print_stats.info.current_layer`, which Rinkhals fills in), saves one picture from the snapshot address at each layer change and builds the video with `ffmpeg` at the end | The live view stays. You must write and run the script. The head is photographed where it is, unless the slicer's G-code parks it (the firmware's own parking, chapter 27, only works with the stock timelapse). |
| B. Set camera 0 to `Anycubic` in the Rinkhals app | The stock path of this chapter can run (the `gkcam` description above): `TIMELAPSE E=1` plus the arming by the app (`openDelayCamera`) | No Mainsail live view. Whether the touchscreen sends the arming message was not found, so it may not work at all. |
| C. Two cameras | Camera 0 left to `gkcam` (`Anycubic`), a second USB camera for `mjpg_streamer` | Needs a second camera and a free USB port (or a hub). |

Option A is the one that does not depend on the Anycubic software and keeps working with Rinkhals. Not tested.

---

## The web request `timelapse/get_config`

The touchscreen reads the state of the function with the web request `timelapse/get_config`. It returns one value, `enable` (1 = timelapse function on, 0 = off), which is the same flag `TIMELAPSE E=` sets. The companion request `timelapse/set_config` takes `enable` (0 or 1) and sets that flag; any other value leaves it unchanged. No other setting (picture position, type) can be read or written this way.

---

### To verify (not tested on a printer)

- The `gkcam` description comes from reading its code (stripped program, no names). The video naming (`<name>_NN.mp4`, first free number 01–99) and the arming message (`video_stream_request` with `method` `openDelayCamera` and `params.filepath`) were traced in the code; the framing of the link between `gkapi` and `gkcam` (address, port, any header around the JSON) and what exactly the 20-entry cleanup counts (all files or only videos) were not.
- Whether the touchscreen actually sends `openDelayCamera` before each print (it is the only thing found that can arm the timelapse; `gkapi` / `K3SysUi` were not searched).
- Whether `M84` really follows the final picture in the "type 1" end sequence.
- Whether any slicer profile calls `SNAPSHOT` / `TIMELAPSE` in the G-code (none of the 17 sample files does; they only carry `; timelapse_type = 0`).
- The `_disconnect` handler of the timelapse module (not examined).

---

*End of chapter 27.*


---

# Chapter 28 — Toolhead and motion limits: `G4`, `M400`, `SET_VELOCITY_LIMIT`, `RESET_VELOCITY_LIMIT`, `SAVE_VELOCITY`, `M201`, `M203`, `M204`, `M205`

This last chapter covers the commands that **wait** (`G4`, `M400`) and the commands that set the printer's **speed and acceleration limits** (`SET_VELOCITY_LIMIT`, `RESET_VELOCITY_LIMIT`, `SAVE_VELOCITY`, `M201`, `M203`, `M204`, `M205`). Slicers write the `M20x` lines at the start of a print and `M204` before many features (outer wall, infill, first layer…), so they are in almost every G-code file.

This firmware does **not** behave like standard Klipper or Marlin for these commands. Read the three rules below before using them.

| Command | What it does |
|---|---|
| `G4 [P<ms>] [S<s>]` | Pause the motion queue for a time |
| `M400` | Wait until all queued moves have really finished |
| `SET_VELOCITY_LIMIT …` | Change several limits (speed, acceleration, corner speed…) |
| `RESET_VELOCITY_LIMIT` | Put every limit back to the configuration-file value |
| `SAVE_VELOCITY [NAME=…]` | Stage the current limit(s) for `SAVE_CONFIG` |
| `M201 X Z E` | Maximum-acceleration "limit" values (X → `max_accel_limit`, Z, E) |
| `M203 X Z E` | Maximum speeds (X → `max_velocity`, Z, E) |
| `M204 S` / `M204 P T` | Print/travel acceleration (`max_accel`) |
| `M205 X E` | Corner ("jerk") speeds |

## The limits and their names

The printer keeps these values. The same value has a Klipper-style name (`SET_VELOCITY_LIMIT`) and sometimes a Marlin-style one (`M20x`). **Stock value** = the stock `printer.cfg` `[printer]` section; the live printer used for the checks in this chapter reports the same numbers.

| Limit (name in status) | `SET_VELOCITY_LIMIT` parameter | M-code | Stock value | What it limits |
|---|---|---|---|---|
| `max_velocity` | `VELOCITY` | `M203 X` | 600 mm/s | Top speed of any move |
| `max_z_velocity` | `Z_VELOCITY` | `M203 Z` | 20 mm/s | Top speed of the Z axis |
| `max_e_velocity` | `E_VELOCITY` | `M203 E` | 600 mm/s | Top speed of the extruder |
| `max_accel` | `ACCEL` | `M204` | 10 000 mm/s² | Acceleration of X/Y moves |
| `max_accel_limit` | `ACCEL_LIMIT` | `M201 X` | 10 000 mm/s² | Second X/Y acceleration value (see notes) |
| `max_z_accel` | `Z_ACCEL` | — | 3 000 mm/s² | Acceleration of the Z axis |
| `max_z_accel_limit` | `Z_ACCEL_LIMIT` | `M201 Z` | 3 000 mm/s² | Second Z acceleration value |
| `max_e_accel` | `E_ACCEL` | — | 10 000 mm/s² | Acceleration of the extruder |
| `max_e_accel_limit` | `E_ACCEL_LIMIT` | `M201 E` | 10 000 mm/s² | Second extruder acceleration value |
| `square_corner_velocity` | `SQUARE_CORNER_VELOCITY` | `M205 X` | 20 mm/s | Speed through a 90° corner (X/Y) |
| `instantaneous_corner_velocity` | `SQUARE_CORNER_VELOCITY_E` | `M205 E` | **1 mm/s** (see below) | Corner speed for extruder-driven changes |
| `minimum_cruise_ratio` | `MINIMUM_CRUISE_RATIO` | — | see below | Share of a move kept at cruise speed |
| `max_accel_to_decel` | `ACCEL_TO_DECEL` | — | 10 000 mm/s² | Acceleration used to slow down |

All of these appear in the `toolhead` object of the printer status (for example through the web request `/printer/objects/query?toolhead`), so you can always check which value is really active.

### Rule 1 — the configuration file is a ceiling

When the printer starts it remembers the values of the `[printer]` section as the **original configuration values**. A command can **lower** a limit, never raise it: any value above the original is silently replaced by the original. Example with the stock values: `M204 S15000` ends up as 10 000; `SET_VELOCITY_LIMIT VELOCITY=800` ends up as 600; `M203 Z30` ends up as 20.

### Rule 2 — a limit you do not mention goes back to the configuration value

In standard Klipper a parameter you leave out keeps its current value. **Here it does not.** In `SET_VELOCITY_LIMIT`, `M201`, `M203` and `M205`, **every limit of the command that is not given (or is given as 0, a negative number or text) is reset to its original configuration value**. Some consequences:

- `M203 X300` sets the speed limit to 300 **and resets the Z and E speed limits** to 20 and 600.
- `SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=9` sets the corner speed to 9 **and resets the acceleration, speeds and everything else to the configuration values** — including an acceleration set a moment earlier by `M204`.
- `SET_VELOCITY_LIMIT` with no parameter at all is therefore a (nearly complete) reset, not a status request.

(`M204` is the exception: it has a single limit, and a line without `S`, `P` or `T` changes nothing.)

### Rule 3 — silence

None of these commands prints anything on the console, even when a value was ignored or replaced. A wrong number is only written to the printer's log. Check what is really active in the `toolhead` status (see above).

### The `_E` corner speed is 1 mm/s, not 20

The stock `printer.cfg` has a line `square_corner_velocity_e: 20`. **The firmware does not read that key.** The value behind `M205 E` and `SQUARE_CORNER_VELOCITY_E` is the Klipper setting `instantaneous_corner_velocity`, which defaults to **1** when absent. The live printer confirms it (`instantaneous_corner_velocity: 1`). Because of rule 1 this value can never be raised above 1 by a command; the usual slicer line `M205 X20.00 Y20.00 Z20.00 E20.00` therefore changes the X/Y corner speed (to 20) and leaves the other at 1.

## How the limits are used (what was found)

- **Normal X/Y moves**: speed ≤ `max_velocity`, acceleration = `max_accel`.
- **Moves with a Z part**: the Z axis may not exceed `max_z_velocity` and **the smaller of** `max_z_accel` and `max_z_accel_limit`, scaled by the share of the move that is vertical.
- **Extruder**: `max_e_velocity` and the E acceleration values limit extrusion (traced less closely; to verify).
- **Corners**: the speed through a corner is derived from `square_corner_velocity` and `max_accel` (Klipper's "junction deviation": `0.414 × square_corner_velocity² ÷ max_accel`). A **lower** acceleration therefore raises the speed allowed in corners, a higher one lowers it (the allowance is recomputed whenever `max_accel` changes).
- **Slowing down**: the deceleration allowance (`max_accel_to_decel`) is always recomputed as `(1 − minimum_cruise_ratio) × max_accel`, so changing `M204` also changes it.
- **`*_limit` values for X/Y**: `max_accel_limit` (`M201 X`) is stored and reported in the status, but the move planner was only found to use `max_accel` for X/Y moves (to verify). The Z limit value is used (see above).

### Two extra limits that apply on top (not G-code commands)

The `[printer]` section also contains two features that Anycubic added and that silently change speeds, whatever the G-code says. They only ever **reduce** a speed or acceleration; they are read from the configuration, not changed by the commands of this chapter.

**1. Acceleration and speed by height** — `kin_param_limit_hight` (stock: `1_600_10000,250_600_2000,350_600_1000`). Each group is `height_speed_acceleration` (mm, mm/s, mm/s²). For every move the firmware looks at the **Z height of the head** (start or end point of the move; which one was not determined) and limits the move's speed and acceleration by linear interpolation between the groups:

| Height of the head | Max speed | Max acceleration |
|---|---|---|
| up to 1 mm | 600 mm/s | 10 000 mm/s² |
| 125 mm (interpolated) | 600 mm/s | ≈ 6 000 mm/s² |
| 250 mm | 600 mm/s | 2 000 mm/s² |
| 300 mm (interpolated) | 600 mm/s | 1 500 mm/s² |
| 350 mm and above | 600 mm/s | 1 000 mm/s² |

So on tall prints the acceleration really used is low even if the G-code says `M204 S10000`; `M204` can only lower it further. (Read from the code and the configuration; not measured on a printer.)

**2. Forbidden speed bands on the Y axis** — `velocity_range_limit` (stock: `45_70,75_90,110_135`, in mm/s). For every move the firmware computes the speed the **Y axis** would have at the move's top speed (top speed × the Y share of the move). If that Y speed is **inside** a band (for example 50 mm/s is inside 45–70), the move's top speed is changed to the **nearer edge of the band**: the upper edge (only if the resulting speed does not exceed `max_velocity`), otherwise the lower edge. The purpose is evidently to avoid bed resonances. Consequence: some slicer speeds (for example a 60 mm/s diagonal or Y-direction move) are not what the G-code asks for. A setting `velocity_range_limit_begin_min` (live printer: 20) and a log switch `velocity_range_limit_hit_log` also exist; how `begin_min` is applied was not analysed. A third key, `accel_height_limit` (`250_4000,300_3000,…`), is read at start-up, but no code using it was found (to verify).

---

## `G4` — Pause

### What it does

1. Reads the delay: `P` in **milliseconds**; if `P` gives 0 (missing, 0 or negative) it reads `S` in **seconds**.
2. Lets the printer finish **planning** every move already received (the planner must now plan the last move so that it ends at standstill), then adds a pause of that length to the move queue. The next move starts only after the pause.

Heaters, fans and the rest keep running normally during a pause. If both `P` and `S` are given, `P` is used.

### Syntax

```
G4 [P<milliseconds>] [S<seconds>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `P` | Pause in milliseconds | 0 or more (a negative value is logged and read as 0) | 0 |
| `S` | Pause in seconds, used only when `P` gives 0 | 0 or more | 0 |

### Examples

```
G4 P500
```
Pause 0.5 s.

```
G4 S2
```
Pause 2 s.

```
G4 S0
```
No pause, but the planner must finish the moves before it (see below). This line appears 751 times in the sample G-code files of the printer.

### Notes and cautions

- `G4` pauses the **motion queue**, not necessarily the real clock at the moment the line is read: moves received earlier may still be running. To be sure every move has physically ended, use `M400`.
- `G4 S0` / `G4 P0` is not a no-op: it forces a planning boundary, so the moves before it are decelerated to a stop at their end (standard Klipper behaviour; it was not tested on this firmware).
- A text instead of a number is read as 0.

---

## `M400` — Wait for the moves to finish

### What it does

Completes the planning of all queued moves, then **waits** (checking every 0.1 s) until the printer has actually executed them. Nothing is printed. Use it before something that must happen after the head has stopped: a camera picture, a door check, a macro that reads a position.

### Syntax

```
M400
```

No parameters.

### Examples

```
G1 X100 Y100 F6000
M400
SNAPSHOT V=1
```
Move, wait until the head has arrived, then take a picture (the timelapse code uses exactly this idea).

### Notes and cautions

- Unlike `G4 S0`, `M400` really holds the G-code reading until the moves are done. It does not wait for temperatures (use `M109`/`M190`/`TEMPERATURE_WAIT`).
- It does not check that the moves succeeded.

---

## `SET_VELOCITY_LIMIT` — Change the limits

### What it does

1. Reads the 13 parameters below. A value that is missing, 0, negative or not a number counts as "not given".
2. If `ACCEL_TO_DECEL` is given, it is stored as the requested deceleration allowance.
3. If `MINIMUM_CRUISE_RATIO` is not given but `ACCEL_TO_DECEL` is, the cruise ratio is **computed**: `1 − min(1, ACCEL_TO_DECEL ÷ acceleration)`, where the acceleration is `ACCEL` if given, otherwise the current `max_accel`. A computed ratio of exactly 0 is not stored (the old ratio stays).
4. Every speed, acceleration and corner-speed limit is set to the value given **if it is above 0 and not above the original configuration value**; **otherwise to the original configuration value** (rule 2).
5. If a cruise ratio was given or computed (and is not 0) it is stored.
6. The corner allowance and the deceleration allowance are recomputed (the latter as `(1 − ratio) × max_accel`).

Nothing is printed; the new values are visible in the `toolhead` status.

### Syntax

```
SET_VELOCITY_LIMIT [VELOCITY=<mm/s>] [Z_VELOCITY=<mm/s>] [E_VELOCITY=<mm/s>]
                   [ACCEL=<mm/s²>] [ACCEL_LIMIT=<mm/s²>]
                   [Z_ACCEL=<mm/s²>] [Z_ACCEL_LIMIT=<mm/s²>]
                   [E_ACCEL=<mm/s²>] [E_ACCEL_LIMIT=<mm/s²>]
                   [SQUARE_CORNER_VELOCITY=<mm/s>] [SQUARE_CORNER_VELOCITY_E=<mm/s>]
                   [MINIMUM_CRUISE_RATIO=<0–1>] [ACCEL_TO_DECEL=<mm/s²>]
```

### Parameters

| Parameter | Meaning | Values | If not given |
|---|---|---|---|
| `VELOCITY` | Top speed | above 0, up to the config value (600) | back to config value |
| `Z_VELOCITY` | Z top speed | up to 20 | back to config value |
| `E_VELOCITY` | Extruder top speed | up to 600 | back to config value |
| `ACCEL` | Acceleration | up to 10 000 | back to config value |
| `ACCEL_LIMIT` | Second X/Y acceleration value | up to 10 000 | back to config value |
| `Z_ACCEL` | Z acceleration | up to 3 000 | back to config value |
| `Z_ACCEL_LIMIT` | Second Z acceleration value | up to 3 000 | back to config value |
| `E_ACCEL` | Extruder acceleration | up to 10 000 | back to config value |
| `E_ACCEL_LIMIT` | Second extruder acceleration value | up to 10 000 | back to config value |
| `SQUARE_CORNER_VELOCITY` | Corner speed (X/Y) | 0 or more, up to 20 | back to config value |
| `SQUARE_CORNER_VELOCITY_E` | Extruder corner speed | up to **1** (see above) | back to config value |
| `MINIMUM_CRUISE_RATIO` | Minimum share of cruise speed | above 0 and below 1 (1 or more is logged and read as 1.0, which would leave **no** slowing allowance: avoid); **0 is ignored** | unchanged (or computed from `ACCEL_TO_DECEL`) |
| `ACCEL_TO_DECEL` | Deceleration allowance, converted to a cruise ratio as in step 3 | above 0 (not capped) | unchanged |

Only these exact parameter names are read; anything else is ignored.

### Examples

```
SET_VELOCITY_LIMIT ACCEL=3000 VELOCITY=200
```
Acceleration 3000 mm/s² and top speed 200 mm/s. Every other limit is **reset** to its configuration value.

```
SET_VELOCITY_LIMIT VELOCITY=300 Z_VELOCITY=10 E_VELOCITY=100 ACCEL=5000 Z_ACCEL=1000 E_ACCEL=2000 SQUARE_CORNER_VELOCITY=10
```
To change several limits and keep the others lower than the ceiling, give them all in one command.

```
SET_VELOCITY_LIMIT
```
Resets every limit (but not the cruise ratio) to the configuration values. It does not display anything.

### Notes and cautions

- **Order matters.** A `M204 S500` followed by `SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=9` ends with an acceleration of **10 000**, not 500. This exact two-line pair is present in the sample G-code files of the printer (first-layer start, e.g. `ACE 2 Pro Lock Helper_PLA_0.2_7m31s.gcode`, line 179–180). If that reading is right, the first layer's low acceleration is cancelled by the next line. **Not tested on a printer** — see "To verify" for a harmless check.
- The resonance test (chapter 13) calls `SET_VELOCITY_LIMIT ACCEL=… MINIMUM_CRUISE_RATIO=0` and restores the values afterwards; with this firmware the ratio `0` is ignored and the acceleration cannot exceed the configuration value.
- `MINIMUM_CRUISE_RATIO` and `ACCEL_TO_DECEL` are only meaningful together with a coherent `ACCEL`; with the stock values (`max_accel_to_decel` = `max_accel` = 10 000) the ratio is 0 — but see the observation in "To verify".
- The result is **not saved**. It lasts until a restart, `RESET_VELOCITY_LIMIT` or the end of a print that resets it (`CANCEL_PRINT` and `G9113` do).

---

## `RESET_VELOCITY_LIMIT` — Back to the configuration values

### What it does

Sets all speeds, accelerations, `ACCEL_TO_DECEL` and both corner speeds back to the original configuration values (the ceilings), then recomputes the corner and deceleration allowances. A line is written in the log only; nothing on the console.

**The `minimum_cruise_ratio` is not reset** by this command (it keeps its current value).

### Syntax

```
RESET_VELOCITY_LIMIT
```

### Parameters

None.

### Examples

```
RESET_VELOCITY_LIMIT
```
Use it at the end of a macro that lowered the limits (calibration, a slow first layer…).

### Notes and cautions

- Called by the printer itself in the print-cancel sequence (chapter 21) and in `G9113` (chapter 2).
- It cancels the effect of `M201`, `M203`, `M204`, `M205` and `SET_VELOCITY_LIMIT` — but not of the height and Y-band limits described above.

---

## `SAVE_VELOCITY` — Stage the current limit(s) for the configuration file

### What it does

Takes the **current** value of the limit named by `NAME` and stores it in the `[printer]` section of the staged configuration (formatted with two decimals), under the Klipper key name (`max_velocity`, `max_accel`, …). Nothing is written to disk yet: like other staged settings (chapter 17) it only becomes permanent after **`SAVE_CONFIG`**, and it is read at the next start.

Because the current values can never be above the ceiling, `SAVE_VELOCITY` can only **keep or lower** the defaults of the printer; after a restart the saved value becomes the new original value (the new ceiling). Raising it again would need an edit of the configuration (inferred).

If `NAME` is not one of the names below (or is empty), nothing is saved and the current values of the 12 limits are **printed** in one line instead (`max_velocity: …, max_z_velocity: …, … square_corner_velocity: …, instantaneous_corner_velocity: …`).

### Syntax

```
SAVE_VELOCITY [NAME=<name>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `NAME` | Which limit to stage (not case-sensitive) | `VELOCITY`, `Z_VELOCITY`, `E_VELOCITY`, `ACCEL`, `ACCEL_LIMIT`, `Z_ACCEL`, `Z_ACCEL_LIMIT`, `E_ACCEL`, `E_ACCEL_LIMIT`, `SQUARE_CORNER_VELOCITY`, `SQUARE_CORNER_VELOCITY_E` (saved as `instantaneous_corner_velocity`), `ACCEL_TO_DECEL`, or `ALL` (all 12) | `VELOCITY` |

`MINIMUM_CRUISE_RATIO` has no name here and is never saved.

### Examples

```
SAVE_VELOCITY NAME=ACCEL
SAVE_CONFIG
```
Make the present acceleration the new default.

```
SAVE_VELOCITY NAME=?
```
Show the current limits on the console (any unknown name does this).

### Notes and cautions

- A bare `SAVE_VELOCITY` saves only `max_velocity`, not everything.
- `ACCEL_TO_DECEL` saves the *requested* deceleration value, not the value derived from the cruise ratio.
- As for any staged value, whether `SAVE_CONFIG` really applies it without a restart was not confirmed on this printer (chapter 17 and the notes on `SAVE_CONFIG`).

---

## `M201` — Maximum acceleration "limit" values

### What it does

Sets the three `*_accel_limit` values from the parameters `X` (→ `max_accel_limit`), `Z` (→ `max_z_accel_limit`) and `E` (→ `max_e_accel_limit`), by the rules above (≤ configuration value; a missing or non-positive one is reset). Recomputes the corner and deceleration allowances. Prints nothing.

`Y` is read by nobody: only `X` stands for the X/Y pair.

### Syntax

```
M201 [X<mm/s²>] [Z<mm/s²>] [E<mm/s²>]
```

### Parameters

| Parameter | Meaning | Values | If not given |
|---|---|---|---|
| `X` | `max_accel_limit` (X/Y) | above 0, up to 10 000 | back to config value |
| `Z` | `max_z_accel_limit` | up to 3 000 | back to config value |
| `E` | `max_e_accel_limit` | up to 10 000 | back to config value |

### Examples

```
M201 X10000 Y10000 Z3000 E10000
```
The line written by the slicer (it appears in the printer's sample files). `Y` is ignored; the values equal the ceilings, so nothing changes.

```
M201 X10000 Y10000 Z1500 E5000
```
The Z limit becomes 1 500 (and `Z` moves are then limited by it), the extruder limit 5 000.

### Notes and cautions

- This is **not** the Marlin "maximum acceleration" of the machine; `M204` sets `max_accel`.
- The effective limit on Z is the smaller of `max_z_accel` and `max_z_accel_limit`.

---

## `M203` — Maximum speeds

### What it does

Sets `max_velocity` (`X`), `max_z_velocity` (`Z`) and `max_e_velocity` (`E`) by the same rules (≤ configuration value; missing or non-positive → reset). Prints nothing. `Y` is ignored.

### Syntax

```
M203 [X<mm/s>] [Z<mm/s>] [E<mm/s>]
```

### Parameters

| Parameter | Meaning | Values | If not given |
|---|---|---|---|
| `X` | `max_velocity` — top speed of X/Y moves | above 0, up to 600 | back to 600 |
| `Z` | `max_z_velocity` | up to 20 | back to 20 |
| `E` | `max_e_velocity` | up to 600 | back to 600 |

### Examples

```
M203 X600 Y600 Z20 E600
```
The usual slicer line; changes nothing in stock configuration.

```
M203 X600 Y600 Z15 E80
```
Z speed limited to 15 mm/s and **the extruder limited to 80 mm/s** (present in some of the sample files): quick filament moves (retractions, purges) are slower.

### Notes and cautions

- Because of rule 2, a line like `M203 X300` also resets Z and E.
- The speed bands of the Y axis (see above) still apply on top.

---

## `M204` — Acceleration

### What it does

1. If `S` is given, uses it as the acceleration. Otherwise it reads `P` (print) and `T` (travel): if both are given, **the smaller** is used; if only one, that one. (`R`, the Marlin retract acceleration, is ignored.)
2. If neither `S`, `P` nor `T` is given, writes `Invalid M204 command: <the line>` **in the log** (not on the console) and changes nothing.
3. The value is set as `max_accel` **if it is above 0 and not above the configuration value; otherwise `max_accel` goes back to the configuration value** (so `M204 S0` or `M204 S-1` resets it, and `M204 S20000` gives 10 000).
4. If `max_accel` really changed, the corner and deceleration allowances are recomputed.

Nothing is printed. The other limits are not touched.

### Syntax

```
M204 S<mm/s²>
M204 [P<mm/s²>] [T<mm/s²>]
```

### Parameters

| Parameter | Meaning | Values | Default |
|---|---|---|---|
| `S` | Acceleration (used first if present) | above 0 (up to 10 000; above it is reduced to 10 000; 0 or negative resets to 10 000) | none |
| `P` | Print acceleration, used when `S` is absent | above 0 (a value of 0 or less is logged and read as 0, which resets to 10 000) | none |
| `T` | Travel acceleration, used when `S` is absent | same as `P` | none |

### Examples

```
M204 S5000
```
Acceleration 5 000 mm/s² for all following moves.

```
M204 P2000 T10000
```
Because the **smaller** of `P` and `T` is used, this gives 2 000 for everything, including travel moves — unlike Marlin.

```
M204 S500
```
Slow acceleration, used by the slicers for the first layer.

### Notes and cautions

- The most frequent values in the sample files are `S10000`, `S5000`, `T10000`, `S1000`, `P5000`, `P2000`, `P500`.
- `M204` is overridden by a following `SET_VELOCITY_LIMIT` that does not give `ACCEL` (see that command), and acceleration is further limited at height (see above).
- `M204 R…` alone is "invalid" (only logged).

---

## `M205` — Corner speeds ("jerk")

### What it does

Sets `square_corner_velocity` from `X` and the extruder/instantaneous corner velocity from `E`, by the same rules (≤ configuration value; missing or non-positive → reset). Then recomputes the corner and deceleration allowances. Prints nothing. `Y`, `Z` and every other letter (`J`, `S`, `T`…) are ignored.

### Syntax

```
M205 [X<mm/s>] [E<mm/s>]
```

### Parameters

| Parameter | Meaning | Values | If not given |
|---|---|---|---|
| `X` | `square_corner_velocity` (speed through a 90° corner) | above 0, up to 20 | back to 20 |
| `E` | `instantaneous_corner_velocity` | above 0, up to **1** | back to 1 |

### Examples

```
M205 X9.00 Y9.00 Z3.00 E3.00
```
Lower corner speed 9 (a line found in the sample files); `E3.00` is reduced to 1 (the ceiling), `Y`/`Z` are ignored.

```
M205 X20.00 Y20.00 Z20.00 E20.00
```
Stock values; the net result is X = 20 and E = 1.

### Notes and cautions

- The corner speed is *not* a Marlin "jerk" in the strict sense: it feeds Klipper's junction-deviation formula. A lower value gives lower speed in corners and a neater print at the cost of time.
- Some slicers write these lines at the print start, and some set `SQUARE_CORNER_VELOCITY` by `SET_VELOCITY_LIMIT` instead — see the order warning there.

---

### To verify (not tested on a printer)

All results come from reading the firmware, plus read-only queries of the live printer's status and configuration. Nothing was sent to the printer.

- **Rule 2 in practice.** The firmware reading says that `SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=9` resets the acceleration. A harmless test on an idle printer: read the `toolhead` status; send `M204 S3000`; read it again (`max_accel` 3000?); send `SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=9`; read again (`max_accel` back to 10000? `square_corner_velocity` 9?). Finish with `RESET_VELOCITY_LIMIT`. It would also confirm that the first-layer `M204 S500` of many profiles is cancelled in this firmware.
- **`minimum_cruise_ratio` on the live printer.** The toolhead status shows `minimum_cruise_ratio: 0.5` while the live configuration says `0.05` (and stock `printer.cfg` would give 0). Something changed it after start-up (an `ACCEL_TO_DECEL` command from a calibration or a print?) or the status shows another value than the setting; the cause was not found. A `RESET_VELOCITY_LIMIT` does **not** change it; only a restart would.
- **Whether `max_accel_limit` / `M201 X` has any effect on X/Y moves.** Only the report in the status and the Z/extruder limits were found in use.
- **The extruder limits** (how `max_e_accel`/`max_e_accel_limit` are applied in detail), **the `begin_min` setting of the Y bands**, and **`accel_height_limit`** (no user found).
- That `SAVE_VELOCITY` + `SAVE_CONFIG` takes effect after a restart, and what a printer with already-staged values does at start (chapter 17 covers the open questions about `SAVE_CONFIG`).
- That the unknown-`NAME` message of `SAVE_VELOCITY` appears on the console (it is sent through the command's answer function, with the "log" option off).

---

*End of chapter 28 — the last chapter of this guide.*
