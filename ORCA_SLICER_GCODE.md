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
- Keep your *filament start G-code* (for example `M900 K…`): it runs after the machine start code.

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
CLEAR_PAUSE                       ; forget a leftover "paused" state (G9111 does this too)
M107                              ; part fan off
G90                               ; absolute XYZ
BED_MESH_CLEAR                    ; no mesh while homing / wiping / measuring the Z offset

; --- 2. Start the bed, home, heat the nozzle (all while the bed heats) ---------
M140 S{min(bed_temperature_initial_layer_single, 60)}   ; start bed heating now, no waiting
G28                               ; home X, Y, Z
MOVE_HEAT_POS                     ; Z15, then X457: nozzle beside the bed, drips fall off the plate
M104 S170                         ; wipe temperature
M109 S170

; --- 3. Clean the nozzle on the wiper pad (the pad is off the bed: bed can still be heating)
G9113                             ; reset "nozzle already cleaned", otherwise G9112 does nothing
G9112                             ; wipe (fan 100 %, 3 passes on the pad)
M400
M107
MOVE_HEAT_POS                     ; lift off the pad and park again while the bed finishes

; --- 3b. Bed to preheat temperature, then 1 minute soak right before measuring --
M190 S{min(bed_temperature_initial_layer_single, 60)}   ; wait until the bed is within 2 C of target
G4 P60000                         ; 1 minute heat soak

; --- 4. LeviQ3 automatic Z offset ----------------------------------------------
BED_MESH_CLEAR
LEVIQ_AUTO_ZOFFSET                ; nozzle to 140 C, G28 Z, strain-gauge measurement, saved
M400
G9113                             ; leave the "cleaned" memory reset (G9111 does the same)

; --- 5. Adaptive bed mesh (nozzle is at 140 C, bed at the preheat temperature) --
BED_MESH_CALIBRATE ADAPTIVE=1 ADAPTIVE_MARGIN=5 PROFILE=adaptive
M400

; --- 6. Bed to print temperature, then nozzle ----------------------------------
MOVE_HEAT_POS
M140 S[bed_temperature_initial_layer_single]
TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM={bed_temperature_initial_layer_single - 10}   ; bed within 10 C
M104 S[nozzle_temperature_initial_layer]   ; only now start the nozzle: no long wait at high temperature = no ooze
M190 S[bed_temperature_initial_layer_single]
M109 S[nozzle_temperature_initial_layer]
LEVIQ_TEMP_OFFSET                 ; Z correction for the hot nozzle (the offset was measured at 140 C)
M400

; --- 7. Purge line, starting ON the bed ----------------------------------------
M117                              ; REQUIRED by the firmware: print-start signal (X/Y to SpreadCycle, timer reset)
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
| 2 | `M140` → `G28` → `M104/M109 S170` | The bed warms up while the printer homes and the nozzle heats, parked off the bed. |
| 3 | `G9113`, `G9112`, then `M190` + `G4 P60000` | The wiper pad is off the bed, so the wipe happens while the bed is still heating. `G9112` does nothing when its "already cleaned" memory is set, so it is reset first; it also clears and reloads the mesh and switches the X/Y drivers around the wipe. Then `M190` (only waits until ±2 °C) and the 60 s soak, right before the measurements, so the bed is stable when the Z offset and mesh are taken. |
| 4 | `LEVIQ_AUTO_ZOFFSET` | Cools the nozzle to 140 °C, homes Z, measures twice (and again after a forced wipe if the result looks suspiciously small), then applies and saves the probe `z_offset` (`IS_SAVE=1` is the default and the only reliable mode, ch. 14). |
| 5 | `BED_MESH_CALIBRATE ADAPTIVE=1 …` | Measures only the first-layer area + 5 mm (minimum 100 × 100 mm, ch. 3). The profile is called `adaptive` so the full-bed `default` mesh that `G9112` reloads is not overwritten. The mesh is taken **after** the Z offset, so it is consistent with the new offset. |
| 6 | `M140`, `TEMPERATURE_WAIT`, `M104`, `M190`, `M109`, `LEVIQ_TEMP_OFFSET` | The bed is set first; the nozzle starts once the bed is within 10 °C, so the end of the bed heat-up and the nozzle heat-up overlap but the nozzle never waits long at high temperature (no ooze). Then `M190`/`M109` wait for both, and the thermal Z correction `(T − 140) × 0.06 / 80` is applied. |
| 7 | `M117`, purge line | `M117` is the firmware's print-start signal, not a message (ch. 5); Anycubic's own start code has it right after the preparation. The purge is the stock `UNDERLINE` idea (about 24 mm of filament, 0.3 mm high) but drawn from X30 to X180 on the front edge, entirely on the bed (the stock one starts at X457, outside it). |

### Things to know

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
M104 S0                           ; nozzle heater off
M140 S0                           ; bed heater off

; --- Present the print ------------------------------------------------------------
G90
G1 Z{min(max_layer_z + 30, 500)} F1200   ; lift 30 mm above the print (Z max is 502)
M400
G1 X0 Y420 F9600                  ; head rear-left = bed fully forward, same spot as the cancel parking
M400

; --- Clean up and shut down -------------------------------------------------------
RESET_VELOCITY_LIMIT              ; back to the configured speed/acceleration limits
M107                              ; part-cooling fan off
M84                               ; steppers off (the axes become "not homed")
; ===== end of end G-code =====
```

### Notes

- **Order:** retract, heaters off, lift, move, then fan and motors. The heaters are switched off as soon as the plastic is done, so they cool during the presentation move.
- **Fans:** only the part-cooling fan is under G-code control (`M107`). The hotend fan and the controller fan are automatic and keep running while the nozzle is warm and for 30 s after the motors stop (ch. 9). Nothing to add for them.
- **Presentation position:** Y420 is the same rear position that `CANCEL_PRINT` uses (`MOVE_END_POS`: X0 Y420). On this bed-slinger, a high Y puts the nozzle at the rear of the bed, so the bed comes forward. Change `X0` if you want the head somewhere else (for example `X457`, like the pause position).
- **`M84` un-homes the printer**; the next print starts with `G28` anyway. Z is not held with the motors off: the cancel routine does the same after a 15 mm lift, so this follows the firmware's own behaviour.
- **Stealth mode and pressure advance** are restored by the firmware at the end of a print (`OPEN_STEALTHCHOP`, ch. 26; the end/cancel path also resets `M900 K`), so the end code does not repeat them.
- The stock Anycubic end code is only `M140 S0 / M104 S0 / M107 / M84`; this version adds the retract, the lift, the presentation move and the limit reset.
- To wait for the nozzle/bed to cool before you walk away, add `TEMPERATURE_WAIT SENSOR=extruder MAXIMUM=50` (no time limit, ch. 11).

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
