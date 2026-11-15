# Orca Slicer G-code for the Anycubic Kobra 3 Max (firmware 2.5.2.8)

Machine start, machine end, before-layer-change and layer-change G-code for **Orca Slicer**, built only from the commands documented in [`K3M_gcode_user_guide.md`](K3M_gcode_user_guide.md).

> **Nothing here has been test-run on a printer.** Statements marked **(to verify)** rest on the firmware analysis only. Do a first print with your hand on the power switch, and watch the first start sequence (wipe, Z-offset, mesh) completely.

## Where to paste

Orca Slicer → *Printer settings* → *Machine G-code*:

| Orca field | Block below |
|---|---|
| Machine start G-code | 1 |
| Machine end G-code | 2 |
| Before layer change G-code | 3 |
| Layer change G-code | 4 |

Also check, in the same printer profile:

- *G-code flavor* = **Klipper**.
- *Use relative E distances* = on (the start code uses `M83`).
- *Label objects* = on (Others → "Label objects"), so Mainsail / the touchscreen can cancel single objects.
- *Emit machine limits to G-code* = **off**. This firmware resets every velocity/acceleration limit that a `SET_VELOCITY_LIMIT` / `M201` / `M203` / `M205` does not name back to the config value (guide ch. 28, rule 2), so a partial limit line can silently undo another one.
- The machine start code sets the pressure advance from the filament profile (*Enable pressure advance* + *Pressure advance*, step 7). Do not also put an `M900` in the *filament start G-code*: it runs after the machine start code and would override it.
- **A successful `PA_AUTO_CALIBRATE` overrides this `M900 K` for the next print** (guide ch. 4): the calibrated value is used instead until that print ends. Check `pressure_advance` in the `extruder` status if the value seems to have no effect.

## One-time setup: two macros in the printer configuration

A print file can only run `G`/`T`/`M`+digit lines and a short list of other commands (see "Why most commands are written as `M7000 <command>`" below). Two macros whose names start with `M` + digit get everything else through. **Add them to the printer configuration before using the blocks below**, then reboot or power-cycle the printer (not `RESTART`, ch. 5). Without them the lines `M7000 …` and `M7001 …` are unknown commands.

```
[gcode_macro M7000]
description: Run the command given as parameter (print files only run G/M/T lines)
gcode:
    {{ rawparams|safe }}

[gcode_macro M7001]
description: Show a message in the console
gcode:
    {{ action_respond_info(rawparams, "") }}

[gcode_macro M7002]
description: Show a message followed by the current Z offsets in the console
gcode:
    {{ action_respond_info(rawparams|add:" - G-code Z offset "|add:printer.gcode_move.homing_origin.z|floatformat:4|add:" mm, probe z_offset (config status, may be stale) "|add:printer.configfile.settings.probe.z_offset|floatformat:4, "") }}
```

- `M7000 BED_MESH_CLEAR` runs `BED_MESH_CLEAR`; `M7001 hello` prints `hello` in the console. `action_respond_info` needs the second (unused) argument `""`, the template engine refuses a call with one.
- **Test in this order:** type `M7001 hello` in the console, then run a print file with the three lines `M7001 hello from a file`, `M7000 M115`, `M7000 BED_MESH_OUTPUT`. You should get the message, the firmware line and the mesh answer.
- **Message format rule.** The text after `M7001` must start with a **letter**, with no `(`, `/`, `*` or other symbols before the first word. The line parser reads an `M`/`G` command name up to the first letter, `/` or `*`, so `M7001 (1/8) Reset` becomes a command called `M7001 (1`, which does not exist, and the line is silently ignored (seen on the printer: the messages of the first Orca file never appeared). After the first word, plain text is fine. Also no square brackets (Orca placeholders) and no runs of four spaces.
- The same rule applies to `M7000`: always write the command right after it, `M7000 BED_MESH_CLEAR`.
- **`M7002 <text>`** prints the text, then the Z offsets: the live G-code Z offset (`printer.gcode_move.homing_origin.z`) and the probe `z_offset` as the config status shows it (it can be stale after `SAVE_CONFIG`, ch. 3). It is called after the LeviQ3 measurement and after `LEVIQ_TEMP_OFFSET`, so the console shows the offset before and after the temperature compensation. The same message rule as `M7001` applies (start with a letter). **(to verify)** the `|add` string concatenation and the `printer.…` paths in this pongo2 template were not run; test with `M7002 test` in the console first. If it prints nothing, check `/tmp/gklib.log` (template errors are only logged, ch. 16).
- **(to verify)** `rawparams` and `|safe` come from the guide's description of the template engine and from the firmware code; nothing was run. `|safe` stops the engine from escaping quotes or `<`/`>`; the commands here contain none.

---

## 1. Machine start G-code

```gcode
; ===== Kobra 3 Max - Orca machine start G-code =====

; --- Information the firmware reads from the file (start of line, exact format) ---
; It is needed by BED_MESH_CALIBRATE ADAPTIVE=1 below. Without these 4 lines the
; firmware silently measures the whole bed instead.
; print_bed_min = 0,0
; print_bed_max = 426,420
; first_layer_print_min = {first_layer_print_min[0]},{first_layer_print_min[1]}
; first_layer_print_max = {first_layer_print_max[0]},{first_layer_print_max[1]}

; --- 1. Reset the state --------------------------------------------------------
M7001 Step 1 of 8 - Reset state
M7000 CLEAR_PAUSE                       ; forget a leftover "paused" state (G9111 does this too)
M107                              ; part fan off
G90                               ; absolute XYZ
M7000 BED_MESH_CLEAR                    ; no mesh while homing / wiping / measuring the Z offset

; --- 2. Start the bed, home, heat the nozzle (all while the bed heats) ---------
M7001 Step 2 of 8 - Homing, bed preheat started, nozzle to wipe temperature
M140 S{min(bed_temperature_initial_layer_single, 60)}   ; start bed heating now, no waiting
G28 W                             ; home X, Y, Z with Z measured at the wiper platform (X30 Y404), like the stock G9112 does: the wipe heights assume this Z reference
G1 Z15 F1200                     ; stay 15 mm up
G1 X0 Y440 F9600                  ; heating spot: rear left, left of the wiper pad and behind the bed (the LeviQ3 parking spot, platform_offset_extru)
M104 S170                         ; wipe temperature, heated right next to the pad
M109 S170

; --- 3. Clean the nozzle on the wiper pad (the pad is off the bed: bed can still be heating)
M7001 Step 3 of 8 - Cleaning the nozzle
G9113                             ; reset "nozzle already cleaned", otherwise G9112 does nothing
G9112                             ; wipe (fan 100 %, 3 passes on the pad)
M400
M107
G1 Z15 F1200                     ; lift off the pad
G1 X0 Y440 F9600                  ; back to the rear-left spot while the bed finishes heating

; --- 3b. Bed to preheat temperature, then 1 minute soak right before measuring --
M7001 Step 4 of 8 - Waiting for the bed, then 1 minute heat soak
M104 S140                         ; nozzle cools down to the LeviQ3 probing temperature (extru_end_temp) during the wait
M190 S{min(bed_temperature_initial_layer_single, 60)}   ; wait until the bed is within 2 C of target
G4 P60000                         ; 1 minute heat soak

; --- 4. LeviQ3 automatic Z offset ----------------------------------------------
M7001 Step 5 of 8 - Measuring the Z offset (LeviQ3)
M7000 BED_MESH_CLEAR
M7000 LEVIQ_AUTO_ZOFFSET                ; nozzle to 140 C, G28 Z, strain-gauge measurement, saved
M400
M7002 Z offset after the LeviQ3 measurement
G9113                             ; leave the "cleaned" memory reset (G9111 does the same)

; --- 5. Adaptive bed mesh (nozzle is at 140 C, bed at the preheat temperature) --
M7001 Step 6 of 8 - Measuring the adaptive bed mesh
M7000 BED_MESH_CALIBRATE ADAPTIVE=1 ADAPTIVE_MARGIN=5 PROFILE=adaptive
M400

; --- 6. Bed to print temperature, then nozzle ----------------------------------
M7001 Step 7 of 8 - Heating to print temperatures
G1 Z15 F1200
G1 X457 F9600                     ; park beside the bed again
{if bed_temperature_initial_layer_single > 70}
M190 S{bed_temperature_initial_layer_single - 10}   ; hot bed: first wait until it is 10 C below its final temperature
{endif}
M104 S[nozzle_temperature_initial_layer]   ; start the nozzle for the last part of the bed heat-up: no long wait at high temperature = no ooze
M190 S[bed_temperature_initial_layer_single]
M109 S[nozzle_temperature_initial_layer]
M7000 LEVIQ_TEMP_OFFSET                 ; Z correction for the hot nozzle (the offset was measured at 140 C)
M400
M7002 Z offset after the temperature compensation

; --- 7. Purge line, starting ON the bed ----------------------------------------
M7001 Step 8 of 8 - Purge line, starting the print
M117                              ; REQUIRED by the firmware: print-start signal (X/Y to SpreadCycle, timer reset)
{if enable_pressure_advance[initial_extruder]}
M900 K{pressure_advance[initial_extruder]}   ; pressure advance of the Orca filament profile (the smooth time is left unchanged)
M7001 Pressure advance K {pressure_advance[initial_extruder]} from the filament profile
{else}
M7001 Pressure advance not enabled in the filament profile, K left unchanged
{endif}
G90
M83                               ; relative E (LEVIQ_AUTO_ZOFFSET leaves absolute mode on)
G92 E0
G1 Z10 F1200
G1 X30 Y5 F9600                   ; front edge, 30 mm inside the bed
G1 Z0.3 F1200
G1 E10 F240                       ; prime blob (about 4 mm/s of filament)
G1 X180 E14 F1800                 ; 150 mm purge line, 14 mm of filament
G1 E-0.8 F1800                    ; retract
G1 X184 Y8 F6000                  ; short sideways wipe to break the string
G1 Z2 F1200
G92 E0
; ===== end of start G-code =====
```

### What each part does and why

| Step | Command(s) | Reason |
|---|---|---|
| 0 | four `; print_bed_… / first_layer_…` comments | The firmware only builds an adaptive mesh when these four lines exist at the start of a line. The stock Anycubic slicer writes them the same way. The bed lines are fixed to the bed of this printer (426 × 420, from `bed_shape` of the sample files); the first-layer lines come from Orca. |
| 1 | `CLEAR_PAUSE`, `BED_MESH_CLEAR` | Clean state. A leftover mesh would bend the wipe and the Z-offset moves. |
| 2 | `M140` → `G28 W` → move to `X0 Y440` → `M104/M109 S170` | The bed warms up while the printer homes. `G28 W` homes Z at the wiper platform (X30 Y404) instead of the bed center: the wipe macro moves to fixed heights (Z −0.8 at the ends of the pad, Z 1 over the brush) that assume this reference, and the stock `G9112` itself homes with `G28 W` when the printer is not homed. A plain `G28` (Z at the bed center) makes `G9112` skip its own homing, and the nozzle can then pass too high above the brush. The nozzle | then heats to the wipe temperature at the rear-left spot, left of the wiper pad (X42–102) and behind the bed edge (Y420), so drips fall outside the bed. `X0 Y440` is the spot the vendor's own LeviQ3 preheat uses (`platform_offset_extru`, ch. 14). |
| 3 | `G9113`, `G9112`, back to `X0 Y440`, `M104 S140`, `M190` + `G4 P60000` | The wiper pad is off the bed, so the wipe happens while the bed is still heating. `G9112` does nothing when its "already cleaned" memory is set, so it is reset first; it also clears and reloads the mesh and switches the X/Y drivers around the wipe. While the printer waits for the bed (`M190`, only to ±2 °C) and soaks for 60 s, the nozzle target is set to 140 °C (the probing temperature `extru_end_temp`), so it cools during the wait and `LEVIQ_AUTO_ZOFFSET` finds it already there. Soak and cooling happen right before the measurements. |
| 4 | `LEVIQ_AUTO_ZOFFSET` | Cools the nozzle to 140 °C, homes Z, measures twice (and again after a forced wipe if the result looks suspiciously small), then applies and saves the probe `z_offset` (`IS_SAVE=1` is the default and the only reliable mode, ch. 14). |
| 5 | `BED_MESH_CALIBRATE ADAPTIVE=1 …` | Measures only the first-layer area + 5 mm (minimum 100 × 100 mm, ch. 3). The profile is called `adaptive` so the full-bed `default` mesh that `G9112` reloads is not overwritten. The mesh is taken **after** the Z offset, so it is consistent with the new offset. |
| 6 | `M190` (hot beds only), `M104`, `M190`, `M109`, `LEVIQ_TEMP_OFFSET` | A bed above 70 °C is first brought to 10 °C below its final temperature; the nozzle then starts, so the end of the bed heat-up and the nozzle heat-up overlap but the nozzle never waits long at high temperature (no ooze). Then `M190`/`M109` wait for both, and the thermal Z correction `(T − 140) × 0.06 / 80` is applied. |
| 7 | `M117`, `M900 K…`, purge line | `M117` is the firmware's print-start signal, not a message (ch. 5); Anycubic's own start code has it right after the preparation. `M900 K{pressure_advance[initial_extruder]}` applies the *Pressure advance* value of the Orca filament profile, but only when *Enable pressure advance* is ticked there; otherwise nothing is sent and the firmware keeps its own `K` (0.02, restored at the end of every print). Only `K` is sent: the smooth time stays at the firmware value (0.04 s). The purge is the stock `UNDERLINE` idea (about 24 mm of filament, 0.3 mm high) but drawn from X30 to X180 on the front edge, entirely on the bed (the stock one starts at X457, outside it). |

### Things to know

- **Never use `TEMPERATURE_WAIT` in this firmware: it never returns.** The first two versions of step 6 used it and the printer stayed stuck with the nozzle target at 140 °C, even with `MAXIMUM` added (found on the printer, then in the firmware code: the value of `MAXIMUM` is capped at `MINIMUM` by a porting bug, so the wait window is always empty, ch. 11 of the guide). The start code now uses `M190` instead: for a bed above 70 °C it waits until the bed is 10 °C below its final temperature, then starts the nozzle heating, then waits for the bed and the nozzle. For a bed at 70 °C or less the bed is already at (or within 10 °C of) its temperature after the preheat, so the nozzle simply starts first.

- **Cooling to 140 °C:** the value is `extru_end_temp` of the `[leviQ3]` section (stock 140). If you changed it, change the `M104 S140` line too. Passive cooling from 170 °C to 140 °C takes a minute or two; if it is not there when the soak ends, `LEVIQ_AUTO_ZOFFSET` just waits for the rest. To speed it up you can add `M106 S255` before the soak (the firmware turns the fan off itself in `LEVIQ_AUTO_ZOFFSET`).
- **Heating spot `X0 Y440`:** it is outside the bed on the left of the wiper pad. If your wiper or a cable is in the way, change both `G1 X0 Y440` lines (X up to about 30 is still left of the pad; the fine-wipe spot of LeviQ3 is near X30 Y420, so do not park there).

- **Why most commands are written as `M7000 <command>`: the print reader skips lines.** Found in the firmware (disassembly of the print-file reader, 2026-11-14) and consistent with every test on the printer: a line of a print file is only executed if it starts with `G`+digit, `T`+digit or `M`+digit (the exact line `M84` is refused), or with one of 12 prefixes (`SNAPSHOT`, `SET_IDLE_TIMEOUT`, `SET_VELOCITY_LIMIT`, `SET_PRESSURE_ADVANCE`, `SAVE_GCODE_STATE`, `RESTORE_GCODE_STATE`, `TUNING_TOWER`, `SHAPER_CALIBRATE`, `EXCLUDE_OBJECT_DEFINE`, `EXCLUDE_OBJECT_START`, `EXCLUDE_OBJECT_END`, `EXCLUDE_OBJECT`). **Every other line is silently dropped, with no error**: `BED_MESH_CALIBRATE`, `BED_MESH_CLEAR`, `LEVIQ_*`, `TEMPERATURE_WAIT`, `CLEAR_PAUSE`, `RESET_VELOCITY_LIMIT`, `ECHO`, `MOVE_HEAT_POS`, any custom macro. That is why your macro tests did nothing. Stock files only use lines that pass (`G9111` does the leveling work internally). A macro whose name starts with `M` + digit (like the stock `M486`) passes the filter, so the two macros of the one-time setup (`M7000`, `M7001`) are the way in. **(to verify on the printer: the macros were not run yet.)**
- **Console messages** come from `M7001 <text>` (`action_respond_info` does reach the console). `ECHO` and `STATUS` only write to the log. There is no known command to show text on the printer's display; `M117` ignores its text.
- **Square brackets are Orca placeholders.** `[...]` in any custom G-code, comments included, is expanded by Orca and makes the slicer refuse ("Expecting a legacy variable expansion format"). Do not put them in comments or text.
- **Do not turn the automatic Z offset off** (`LEVIQ_AUTO_ZOFFSET_ON_OFF ON_OFF=0`) while keeping this start code: `LEVIQ_TEMP_OFFSET` would then add its correction to the saved offset at every print. With the measurement on, `LEVIQ_AUTO_ZOFFSET` zeroes the offsets first, so nothing accumulates.
- **`LEVIQ_AUTO_ZOFFSET` and `LEVIQ_TEMP_OFFSET` both run `SAVE_CONFIG`** (on this printer the values go to `printer_mutable.cfg`). That is the same as the stock `G9111`, but it means the config file is rewritten at each print.
- **`LEVIQ_TEMP_OFFSET` takes the print temperature from the file** (header `; nozzle_temperature =` / `; first_layer_temperature =` or an `M109 S…` line). The `M109 S…` line of step 6 is there, so it is found **(to verify which source wins)**.
- **The file has to be started as a print job** (touchscreen, Mainsail "Print"), not pasted line by line in a console: the first-layer information is only collected while a print runs.
- **Bed temperature during the mesh** is the preheat (60 °C, or lower if your print temperature is lower). The bed is then heated further for ABS / PETG-type temperatures. If your mesh looks off at 90–100 °C, raise the `60` in step 2 (the mesh is measured at that temperature).
- **Purge line outside the adaptive mesh.** At Y5 the purge line is usually outside the small adaptive mesh. Klipper-style meshes use the nearest edge value there, which is a reasonable approximation for a purge line. If you want the purge exactly under a measured area, move it next to your print (use `first_layer_print_min`) or measure a full mesh.
- **Purge line collides with a skirt/brim at the front edge?** Change `Y5` (and the wipe `Y8`) to a larger value; the line sits at the front edge of the bed, the bed moves in Y.
- If `M104 S…` / `M140 S…` in this start code makes Orca skip its own heating commands, nothing is lost: all the needed temperatures are set here.
- **Fallback without the four comment lines:** replace the mesh line with explicit limits, for example `BED_MESH_CALIBRATE MESH_MIN=… MESH_MAX=… PROBE_COUNT=5,5 PROFILE=adaptive`, using `{first_layer_print_min[0]}`-style values clamped to 10…410.

---

## 2. Machine end G-code

```gcode
; ===== Kobra 3 Max - Orca machine end G-code =====
M400                              ; finish the last moves
M83                               ; relative E for the retract
G1 E-1.5 F1800                    ; small retract (Orca may already have retracted)

; --- Cool down -------------------------------------------------------------------
M7001 Step 1 of 3 - Print finished: heaters off
M104 S0                           ; nozzle heater off
M140 S0                           ; bed heater off

; --- Present the print ------------------------------------------------------------
M7001 Step 2 of 3 - Presenting the print
G90
G1 Z{min(max_layer_z + 30, 500)} F1200   ; lift 30 mm above the print (Z max is 502)
M400
G1 X0 Y420 F9600                  ; head rear-left = bed fully forward, same spot as the cancel parking
M400

; --- Clean up and shut down -------------------------------------------------------
M7001 Step 3 of 3 - Fan and steppers off
M7000 RESET_VELOCITY_LIMIT              ; back to the configured speed/acceleration limits
M107                              ; part-cooling fan off
M84                               ; steppers off (the axes become "not homed")
; ===== end of end G-code =====
```

### Notes

- **Order:** retract, heaters off, lift, move, then fan and motors. The heaters are switched off as soon as the plastic is done, so they cool during the presentation move.
- **Fans:** only the part-cooling fan is under G-code control (`M107`). The hotend fan and the controller fan are automatic and keep running while the nozzle is warm and for 30 s after the motors stop (ch. 9). Nothing to add for them.
- **Presentation position:** Y420 is the same rear position that `CANCEL_PRINT` uses (the stock `MOVE_END_POS` macro goes to X0 Y420; it is not called here because macros do not run from a print file). On this bed-slinger, a high Y puts the nozzle at the rear of the bed, so the bed comes forward. Change `X0` if you want the head somewhere else (for example `X457`, like the pause position).
- **`M84` un-homes the printer**; the next print starts with `G28` anyway. Z is not held with the motors off: the cancel routine does the same after a 15 mm lift, so this follows the firmware's own behaviour.
- **Stealth mode and pressure advance** are restored by the firmware at the end of a print (`OPEN_STEALTHCHOP`, ch. 26; the end/cancel path also resets `M900 K`), so the end code does not repeat them.
- The stock Anycubic end code is only `M140 S0 / M104 S0 / M107 / M84`; this version adds the retract, the lift, the presentation move and the limit reset.
- `TEMPERATURE_WAIT` cannot be used to wait for the cool-down (it never returns, see the start-code notes). To wait for the nozzle to cool, `M109 S50` followed by `M104 S0` should work (a cooling `M109`/`M190` waits until the temperature is within 2 °C of the new target, ch. 11); not tested.

---

## 3. Before layer change G-code

```gcode
; BEFORE_LAYER_CHANGE [layer_num] @ [layer_z]mm
```

## 4. Layer change G-code (after the layer change)

```gcode
; AFTER_LAYER_CHANGE [layer_num] @ [layer_z]mm
```

### Why these two lines and nothing more

- They are exactly the tags used by Anycubic's own profile and sample files (`; BEFORE_LAYER_CHANGE 0 @ 0.2mm`, `; AFTER_LAYER_CHANGE 0 @ 0.2mm`). The firmware uses the **last** `; AFTER_LAYER_CHANGE <n> @ <z>mm` line of the file as one source of the total layer count (`n + 1`); Orca's own `;LAYER_CHANGE` comment drives the current layer (ch. 23).
- **No `SET_PRINT_STATS_INFO`.** On a Rinkhals printer the Moonraker plugin overwrites `print_stats.info` with the touchscreen's layer numbers, so the command would bring nothing (ch. 23). On a stock-software printer add this to the start code, after step 7: `SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]`, and to block 4: `SET_PRINT_STATS_INFO CURRENT_LAYER={layer_num + 1}` (total first, because a new total resets the current layer). **(to verify)**
- **No `M117`, `M75`, `M204`, `M205` or `SET_VELOCITY_LIMIT`** in the layer-change code: `M117` resets the print timer and switches the motors to SpreadCycle at every call, and the limit commands reset every limit that is not named (ch. 5, ch. 28).
- Moves and extrusion belong to Orca (retraction, Z hop); the layer-change blocks stay comment-only so that they cannot disturb them.
