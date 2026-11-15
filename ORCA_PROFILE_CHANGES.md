# Orca Slicer profiles for the Kobra 3 Max: what was changed from the stock profiles

The three user profiles used for the Kobra 3 Max, compared with the system profiles they inherit from. Every customized setting is listed, not only the ones linked to precision. The machine **start / end G-code** is left out: it is documented in [`ORCA_SLICER_GCODE.md`](ORCA_SLICER_GCODE.md).

| User profile | Inherits from (stock) |
| --- | --- |
| Printer: `Kobra 3 Max - Network` | `Anycubic Kobra 3 Max 0.4 nozzle` |
| Process: `0.20mm Kobra 3 Max 0.4 - Multiboard` | `0.20mm Standard @Anycubic Kobra 3 Max 0.4 nozzle` |
| Filament: `Grossiste 3D PLA Matte @System` | `Generic PLA Matte @System` (there is no plain "Generic PLA" parent: the matte variant is the base) |

**How this list was made (2026-11-15).** The fully resolved settings of the three user profiles were exported with the Orca command line (`--export-settings`) and compared with the export of three stub profiles that only inherit from the stock ones (790 settings each). 38 settings differ (28 process, 5 filament, 5 printer, the printer ones including the two G-code blocks); the list matches the `different_settings_to_system` field that Orca stores in each profile. Nothing else differs, so everything not listed below is stock. The "What it does" column describes the setting, it is not a record of why each value was chosen, except where a note says so.

The filament profile has one column per extruder variant; only the first one, **Direct Drive Standard**, is used by this printer (see section 2).

## 1. Printer profile: `Kobra 3 Max - Network`

| Setting | Stock | Customized | What it does |
| --- | --- | --- | --- |
| Retract when changing layer (`retract_when_changing_layer`) | on | **off** | No retraction at every layer change (retractions only happen on travel moves). |
| Wipe while retracting (`wipe`) | on | **off** | The nozzle does not move along the path while retracting. |
| Print host (`print_host`) | empty | the printer's LAN address | Lets Orca upload files to / talk to the printer over the network (this is the "Network" in the name). The address is not recorded here. |
| Machine start / end G-code | stock Anycubic | custom | See [`ORCA_SLICER_GCODE.md`](ORCA_SLICER_GCODE.md). |

The *Before layer change* and *Layer change* G-code (blocks 3 and 4 of `ORCA_SLICER_GCODE.md`) are identical to the stock profile, so nothing is changed there.

Outside Orca, in the printer's own `printer.cfg`: `square_corner_velocity` was lowered from 20 to **5** mm/s (2026-11-15). It is the ceiling the firmware applies to every corner, whatever the slicer sends, see the corner velocity notes in [`K3M_gcode_user_guide.md`](K3M_gcode_user_guide.md) (`M205`, rules 1 and 2).

## 2. Filament profile: `Grossiste 3D PLA Matte @System`

Only five settings are customized, all of them for the first extruder variant (Direct Drive Standard). Everything else is the stock `Generic PLA Matte @System`.

| Setting | Stock | Customized | What it does |
| --- | --- | --- | --- |
| Nozzle temperature (`nozzle_temperature`) | 220 °C | **210** | Printing temperature. |
| Initial layer nozzle temperature | 220 °C | **210** | First layer temperature, same as the rest. |
| Flow ratio (`filament_flow_ratio`) | 0.98 | **1.01** | Extrusion multiplier, calibrated separately for this filament. |
| Enable pressure advance | off | **on** | The machine start G-code sends `M900 K…` with the value below. |
| Pressure advance (`pressure_advance`) | 0.02 | **0.047** | Pressure advance K, from the automated test (values around 0.05 were judged good). |

> **Calibrate each filament separately.** The flow ratio and the pressure advance above are specific to this filament (brand, colour, even a different spool). For every new filament, create its own filament profile and run both calibrations before printing anything that needs precision: the **flow calibration** (Orca's flow rate test, then enter the result in *Flow ratio*) and the **automatic pressure advance calibration** of the printer, then enter the K value in the profile with *Enable pressure advance* ticked. Do not copy the values from another filament. Note that a successful `PA_AUTO_CALIBRATE` on the printer overrides the `M900 K` of the machine start G-code for the next print (see [`ORCA_SLICER_GCODE.md`](ORCA_SLICER_GCODE.md)), so write the calibrated K into the profile to keep the two in agreement.

**Extruder variants.** The profile has six columns (Direct Drive Standard / High Flow / Extra High Flow, Bowden Standard / High Flow / Extra High Flow). The stock Kobra 3 Max 0.4 profile declares extruder type Direct Drive, variant Standard, brass nozzle, so only the first column is ever used; the others keep their stock values and have no effect.

**What the stock values now do** (the previous version of this profile overrode them, the new one does not):
- **Max volumetric speed** is the stock 11 mm³/s for the Standard variant. Orca slows a feature down when its speed would ask for more plastic: sparse infill at 150 mm/s with 0.45 × 0.2 mm lines is about 13.5 mm³/s and is limited to roughly 120 mm/s, everything else in the process profile is below 11 mm³/s.
- **Minimum layer time** is the stock 4 s with "slow down for layer cooling" on (slowest speed 10 mm/s): very short layers, for example on small test parts, are slowed down to give them cooling time.
- **Filament shrinkage** is 100 % (no compensation), and retraction at layer change and wipe follow the printer profile (both off).

## 3. Process profile: `0.20mm Kobra 3 Max 0.4 - Multiboard`

### Speeds and accelerations

| Setting | Stock | Customized |
| --- | --- | --- |
| Outer wall speed | 150 mm/s | **40** |
| Inner wall speed | 200 mm/s | **60** |
| Top surface speed | 100 mm/s | **50** |
| Sparse infill speed | 300 mm/s | **150** |
| Internal solid infill speed | 150 mm/s | **100** |
| Gap infill speed | 250 mm/s | **40** |
| Outer wall acceleration | 2000 mm/s² | **1000** |
| Inner wall acceleration | 5000 mm/s² | **2500** |
| Top surface acceleration | 1000 mm/s² | **1200** |
| Sparse infill acceleration | 5000 mm/s² | **3000** |
| Internal solid infill acceleration | 5000 mm/s² | **3000** |
| Bridge acceleration | 3000 mm/s² | **1000** |
| Travel acceleration | 10000 mm/s² | **5000** |
| Accel-to-decel factor (`accel_to_decel_factor`) | 50 % | **75 %** |

Why they matter on this printer: the Kobra's `printer.cfg` allows 10000 mm/s² and moves a heavy bed along Y, so high accelerations make the bed ring; and its firmware moves the top speed of any move whose **Y component** falls inside 45–70, 75–90 or 110–135 mm/s to the edge of that band without a message (`velocity_range_limit`). At 40 mm/s the outer wall can never be inside a band; top surface 50, inner wall 60 and the first layer (45) can for moves with a large Y share. The first layer (500 mm/s², 45 mm/s), default acceleration (5000), travel speed (300) and bridge speed (30) are stock.

### Walls, seam and dimensional accuracy

| Setting | Stock | Customized | What it does |
| --- | --- | --- | --- |
| Wall loops (`wall_loops`) | 2 | **3** | Three perimeters. |
| Wall generator (`wall_generator`) | classic | **arachne** | Variable-width walls, handles thin features better. |
| Outer wall line width (`outer_wall_line_width`) | 0.42 mm | **0.39** | Line width of the outer perimeter. A width test (0.36 to 0.40) showed no visible or tactile difference in fit; 0.39 was chosen to keep a little margin on outside dimensions. |
| Seam gap (`seam_gap`) | 10 % | **23 %** | Gap left at the seam between the end and the start of the loop. |
| Smooth (scarf) seam, conditional (`seam_slope_conditional`) | off | **on** | Slopes the seam over several layers on suitable (smooth, non-overhanging) corners. |
| Elephant foot compensation (`elefant_foot_compensation`) | 0.15 mm | **0.2** | Shrinks the first layer's outline to counter the squashed first layer. |
| Convert holes to polyholes (`hole_to_polyhole`) | off | **on** | Round holes are sliced as polygons that print closer to the nominal diameter. |

### Infill, bridges, support, brim

| Setting | Stock | Customized | What it does |
| --- | --- | --- | --- |
| Sparse infill density (`sparse_infill_density`) | 15 % | **20 %** | More infill. |
| Align infill direction to model (`align_infill_direction_to_model`) | off | **on** | The infill pattern turns with the part when it is rotated, not with the bed. |
| Extra bridge layer (`enable_extra_bridge_layer`) | disabled | **external bridges only** | Adds a second layer over bridges that are visible from outside. |
| Don't support bridges (`bridge_no_support`) | off | **on** | Bridges get no support under them. |
| Support type (`support_type`) | tree (auto) | **normal (auto)** | Grid-style supports instead of tree supports. |
| Brim type (`brim_type`) | auto | **none** | No brim. |
| Prime tower (`enable_prime_tower`) | on | **off** | No prime tower (only needed for multi-filament prints). |

### Same as stock but worth knowing

- **Arc fitting is disabled** (the stock value). The firmware cuts every `G2`/`G3` arc into straight segments of about 1 mm, so fitted arcs would coarsen small curves; an earlier override of this profile that enabled it was removed.
- **Resolution 0.012 mm** (stock). Do not set it below 0.012: the firmware halves the print speed for files that declare a smaller resolution (guide, `G1` "halving rule", to verify on a print).
- **Jerk 9** in all the process jerk settings (stock): Orca writes it as `SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=9`, which the `printer.cfg` ceiling of 5 mm/s replaces by 5.
- Wall order inner → outer, layer height 0.2 mm, inner wall width 0.45, default line width 0.42, bridge speed 30, travel speed 300: stock.
- The bed type is not part of these profiles: Orca takes it from the project (default *Cool Plate*, 35 °C bed). Set the plate (for example Textured PEI, 55 °C) in the project before slicing.

## 4. How to check this list again

Export the resolved settings of the user profiles and of three stub profiles that only inherit from the stock ones, and compare them:

```
orca-slicer --datadir <data folder> --load-settings "<machine>.json;<process>.json" --load-filaments "<filament>.json" --arrange 0 --export-settings <out>.json --outputdir <folder> <any 3mf>
```

The stub profiles must be in a copy of the data folder (the command line only accepts profiles it finds in its data folder). The `different_settings_to_system` field of the export lists, per profile, the settings that differ from the system profile and is a quick cross-check.
