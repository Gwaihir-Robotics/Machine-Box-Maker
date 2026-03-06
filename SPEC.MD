# SteelCab: Parametric Steel Cabinet Base Generator

## Software Requirements Specification v0.2

**Author:** Mason / American Stator  
**Version:** 0.2 DRAFT  
**Date:** March 2026

---

## 1. Purpose & Scope

SteelCab is a CLI tool and Python library that programmatically generates:

1. **STEP files** for tube-laser-cut frame members (for OSH Cut / tube laser services)
2. **DXF files** for flat-laser-cut skin panels, doors, shelves, and gussets (for SendCutSend / OSH Cut flat laser)
3. A **bill of materials (BOM)** with cut list, hardware list, and cost estimate
4. Assembly drawings / weld maps (PDF or SVG)

The construction method is **tube frame + flat panel skin**: square steel tube provides the structural skeleton, and laser-cut sheet panels attach to the frame via slots cut in the tube walls, plus bolt-on doors and access panels.

### 1.1 Target Equipment

| Machine | Envelope (L × W × H) | Bay Count | Frame Tube | Notes |
|---------|----------------------|-----------|------------|-------|
| Epoxy powder coating line | 4900 × 2500 × 900mm | 5 | 2" × 2" A500 | Multi-bay, heavy (3000kg total) |
| Stator winding machine | 1220 × 813 × 900mm | 1–2 | 1.5" × 1.5" A513 | Compact, moderate load |

---

## 2. Material Catalog (Hardcoded Stock)

The software ships with a built-in catalog of tube profiles available from OSH Cut and SendCutSend. The user selects profiles by name; the tool validates dimensions and computes structural properties.

### 2.1 Square Tube — A513 (Consumer/Light Structural)

| Size (in) | Wall (in) | Wall (mm) | OD (mm) | Max Length | Weight (lb/ft) | Ix (in⁴) | Source |
|-----------|-----------|-----------|---------|------------|-----------------|----------|--------|
| 0.75 × 0.75 | 0.065 | 1.65 | 19.05 | 235" | 0.60 | 0.023 | OSH Cut |
| 0.75 × 0.75 | 0.083 | 2.11 | 19.05 | 235" | 0.75 | 0.028 | OSH Cut |
| 1 × 1 | 0.049 | 1.24 | 25.40 | 235" | 0.63 | 0.035 | OSH Cut |
| 1 × 1 | 0.065 | 1.65 | 25.40 | 235" | 0.82 | 0.044 | OSH Cut |
| 1 × 1 | 0.083 | 2.11 | 25.40 | 235" | 1.02 | 0.053 | OSH Cut |
| 1 × 1 | 0.12 | 3.05 | 25.40 | 235" | 1.41 | 0.069 | OSH Cut |
| 1.25 × 1.25 | 0.065 | 1.65 | 31.75 | 235" | 1.05 | 0.092 | OSH Cut |
| 1.25 × 1.25 | 0.083 | 2.11 | 31.75 | 235" | 1.31 | 0.111 | OSH Cut |
| 1.25 × 1.25 | 0.12 | 3.05 | 31.75 | 235" | 1.82 | 0.146 | OSH Cut |
| 1.5 × 1.5 | 0.049 | 1.24 | 38.10 | 235" | 0.97 | 0.121 | OSH Cut |
| 1.5 × 1.5 | 0.065 | 1.65 | 38.10 | 235" | 1.26 | 0.154 | OSH Cut |
| 1.5 × 1.5 | 0.083 | 2.11 | 38.10 | 235" | 1.59 | 0.189 | OSH Cut |
| 1.5 × 1.5 | 0.12 | 3.05 | 38.10 | 235" | 2.21 | 0.251 | OSH Cut |

### 2.2 Square Tube — A500 (Structural)

| Size (in) | Wall (in) | Wall (mm) | OD (mm) | Max Length | Weight (lb/ft) | Ix (in⁴) | Source |
|-----------|-----------|-----------|---------|------------|-----------------|----------|--------|
| 1.5 × 1.5 | 0.188 | 4.78 | 38.10 | 235" | 3.28 | 0.347 | OSH Cut |
| 1.5 × 1.5 | 0.25 | 6.35 | 38.10 | 235" | 4.09 | 0.406 | OSH Cut |
| 2 × 2 | 0.12 | 3.05 | 50.80 | 235" / 60" | 3.05 | 0.637 | OSH / SCS |
| 2 × 2 | 0.188 | 4.78 | 50.80 | 235" / 60" | 4.59 | 0.906 | OSH / SCS |
| 2 × 2 | 0.25 | 6.35 | 50.80 | 235" | 5.84 | 1.09 | OSH Cut |
| 2.5 × 2.5 | 0.12 | 3.05 | 63.50 | 235" | 3.86 | 1.27 | OSH Cut |

### 2.3 Flat Sheet (for Panels/Gussets)

| Material | Gauge | Thickness (in) | Thickness (mm) | Max Sheet | Source |
|----------|-------|-----------------|----------------|-----------|--------|
| A1008 Mild Steel | 16 ga | 0.060 | 1.52 | 48×96" | OSH / SCS |
| A1008 Mild Steel | 14 ga | 0.075 | 1.90 | 48×96" | OSH / SCS |
| A1008 Mild Steel | 12 ga | 0.105 | 2.67 | 48×96" | OSH / SCS |
| A1008 Mild Steel | 11 ga | 0.120 | 3.05 | 48×96" | OSH / SCS |
| A1008 Mild Steel | 10 ga | 0.134 | 3.40 | 48×96" | OSH / SCS |

### 2.4 Tube Corner Radius Note

Per A500/A513 specs, outer corner radii on square tube can vary up to 3× wall thickness. The software uses a **conservative default of 2× wall thickness** for modeling corner radii, and this is configurable. This affects slot placement near tube corners — slots must not cross into the corner radius zone.

---

## 3. Architecture: Tube Frame + Flat Panel

### 3.1 Frame Topology

Each cabinet bay is a rectangular box frame built from square tube:

```
        ┌──────── TOP RAIL (front) ────────┐
       ╱│                                 ╱│
      ╱ │ TOP RAIL (left)               ╱ │ TOP RAIL (right)
     ╱  │                              ╱  │
    ┌───┼────── TOP RAIL (back) ──────┐   │
    │   │                             │   │
    │   │  VERTICAL                   │   │ VERTICAL
    │   │  POST (4×)                  │   │ POST
    │   │                             │   │
    │   └─────── BTM RAIL (front) ────│───┘
    │  ╱                              │  ╱
    │ ╱  BTM RAIL (left)              │ ╱  BTM RAIL (right)
    │╱                                │╱
    └──────── BTM RAIL (back) ────────┘
```

**12 members per bay**: 4 vertical posts, 4 top rails, 4 bottom rails.

For multi-bay machines, adjacent bays share vertical posts (8 posts for a 5-bay machine, not 20).

Optional additional members:
- **Mid-height cross rail** (for shelf support)
- **Diagonal bracing** (in back face, for lateral rigidity on tall/narrow bays)
- **Top frame cross members** (spaced along top to support worktop/equipment)

### 3.2 Tube-to-Tube Joints

The tube laser cuts all joints. Three joint types:

#### Cope Joint (default for right-angle T/L connections)
One tube is coped (notched) to wrap around the mating tube:

```
     ┌────┐
     │    │  Rail (coped end)
     │    │
     └─┐┌─┘
       ││
    ┌──┘└──┐
    │      │  Post (continuous)
    │      │
```

- The **posts are continuous** (no cuts along their length at joints) for maximum vertical load capacity
- Rails are coped to wrap the post OD, with the cope profile matching the post face + corner radii
- Cope clearance: 0.005" (0.127mm) per side (tube laser tolerance)

#### Tab-in-Slot Joint (for self-fixturing)
A tab is left protruding from the coped end of the rail; a corresponding slot is cut in the post wall:

```
    Rail end:           Post wall:
    ┌────┐              │  ┌──┐  │
    │ ┌┐ │  tab ──►     │  │  │  │  slot
    │ ││ │              │  └──┘  │
    └─┘└─┘              │       │
```

- Tab width: 0.75" (default, configurable)
- Tab length: equals post wall thickness + 0.030" (protrudes slightly for plug weld)
- Slot width: tab width + 0.010" total clearance (0.005"/side)
- This allows the rail to locate itself on the post before tack welding
- Etch marks on both tube surfaces for alignment verification

#### Miter Joint (for top frame perimeter)
45° miter cuts at corners, with optional alignment tabs:

```
    ╲      ╱
     ╲    ╱
      ╲  ╱  45° miter
       ╲╱
```

- Used at the 4 corners of the top frame for a clean finished edge
- Optional: small alignment tab/slot at the miter face for self-fixturing

### 3.3 Flat Panel Attachment

Flat sheet panels (sides, back, doors, shelves) attach to the tube frame two ways:

#### Method A: Slot-in-Tube (primary)
The tube laser cuts slots in the tube walls. Flat panel edges slide into these slots:

```
    Tube cross-section with slot:
    ┌──────────────────┐
    │  ┌────────────┐  │
    │  │            │  │
    ├──┘            └──┤  ← slot cut through tube wall
    │                  │     (sheet slides in from the side)
    │  │            │  │
    │  └────────────┘  │
    └──────────────────┘
```

- Slot width: sheet thickness + 0.010" clearance
- Slot depth: 0.25" minimum engagement (panel edge extends 0.25" into tube)
- Panels can be plug-welded through the slot or secured with set screws
- Slot locations are on the INNER faces of the tube frame (hidden from exterior)

#### Method B: Bolt-Through (doors and removable panels)
For panels that need to be removable:
- Tube laser cuts clearance holes in the tube wall
- Panel has matching holes
- Secured with M5 or M6 button-head bolts + clip nuts (or weld nuts on tube)

#### Method C: Gusset Bracket (heavy shelves / worktop support)
For load-bearing horizontals:
- L-shaped gusset (laser cut flat, bent 90°) bolts or welds to the tube frame
- Shelf/worktop sits on gusset flanges

### 3.4 Conveyor Mounting Rail

The conveyor rail is integrated into the top-front rail of the frame:

**Option 1: Oversized Top-Front Rail**
Use a larger tube profile for the top-front rail (e.g., 2" × 2" on a machine that otherwise uses 1.5" × 1.5"). The extra face width provides a bolt-through mounting surface for a conveyor bracket.

**Option 2: Welded Mounting Plate**
A flat plate (11ga, 150–200mm tall) is welded to the outside face of the top-front rail, extending above the frame top. Pre-punched bolt holes for conveyor mounting on 100mm spacing.

**Option 3: T-Slot Extrusion Adapter**
Bolt-on aluminum T-slot extrusion (e.g., 40×40mm or 40×80mm Misumi/8020) attached to the tube frame via weld-on brackets. Provides infinite adjustability for conveyor positioning.

### 3.5 Foot Plates

Each vertical post terminates at the bottom with a laser-cut **foot plate** welded to the tube end. The plate serves as a universal mounting interface:

```
    POST (tube)
    ┌──────┐
    │      │
    │      │
    │      │
    └──┬┬──┘
  ┌────┘└────┐
  │ ○  ○  ○  │  ← Foot plate (laser-cut flat, 11ga)
  │ ○  ○  ○  │     Bolt pattern for feet/casters
  └──────────┘
       │
    ┌──┴──┐
    │FOOT │  ← Leveling foot, caster, or combo leveling caster
    └─────┘
```

**Plate design:**
- Square plate, default 120×120mm (configurable, must be larger than tube OD)
- Center hole matches tube interior (allows weld access from inside tube)
- Bolt pattern is configurable to match standard caster/foot mounting holes
- 3 mounting configurations:
  - **Leveling foot**: single center threaded hole (weld nut on underside) for M16/M20 threaded foot
  - **Plate caster**: 4-hole bolt pattern matching caster top plate (e.g., 76×76mm for standard 4"–5" industrial casters)
  - **Combo leveling caster**: 4-hole bolt pattern matching FootMaster/CasterHQ retractable leveling casters

**Foot plate DXF includes:**
- Outer perimeter cut
- Bolt holes (clearance sized)
- Center hole or weld-nut relief (if leveling foot style)
- Engraved post ID (e.g., "FL" for front-left, "S-F02" for shared-front post #2)

### 3.6 Worktop

The tool exports a plain **worktop DXF** — a flat outline sized to the top frame perimeter. This is a reference/starting-point file that the user customizes:

```
    ┌─────────────────────────────────────────────┐
    │                WORKTOP DXF                   │
    │                                              │
    │   ┼ ─ ─ ─ ┼ ─ ─ ─ ┼ ─ ─ ─ ┼ ─ ─ ─ ┼      │  ← cross-member ref lines
    │                                              │
    │   ○        ○        ○        ○        ○      │  ← mounting hole positions
    │                                              │
    └─────────────────────────────────────────────┘
    
    Layers:
      CUT       — outer perimeter
      REFERENCE — cross-member centerlines, post locations
      MOUNTING  — bolt holes aligned with top frame members
```

- Sized to match top frame outer dimensions (+ configurable overhang per side)
- Cross-member positions on a reference layer so the user knows where support exists before adding cutouts
- Mounting holes on a separate layer (user can enable/disable)
- Material is user's choice: for epoxy machines, likely 11ga steel; for winding machines, possibly 3/4" plywood or HDPE
- The DXF is a starting point — the user adds equipment-specific cutouts (screw-drive holes, powder hopper openings, etc.)

### 3.7 Cable Routing

Standard NEC knockout-sized holes are laser-cut in the tube frame for inter-bay wiring:

```
    BOTTOM RAIL (cross-section, looking from front):
    ┌────────────────────────────────────────┐
    │  ┌──────────────────────────────────┐  │
    │  │     ○      ○        ○           │  │  ← knockout holes (top face of bottom rail)
    │  │   22.2mm  28.6mm  34.9mm        │  │     1/2", 3/4", 1" trade sizes
    │  └──────────────────────────────────┘  │
    └────────────────────────────────────────┘
```

**Standard knockout sizes (laser-cut full holes, no holding tabs):**

| Trade Size | Knockout Diameter (in) | Knockout Diameter (mm) | Fits |
|------------|----------------------|----------------------|------|
| 1/2" | 0.875 | 22.2 | 1/2" EMT, liquidtight, cord grips |
| 3/4" | 1.109 | 28.2 | 3/4" EMT, liquidtight |
| 1" | 1.375 | 34.9 | 1" EMT/rigid, larger cord grips |
| 1-1/4" | 1.701 | 43.2 | 1-1/4" conduit (available on request) |

- Holes are cut in the **top face** of bottom rails (accessible from inside the cabinet)
- Placement: configurable per bay (default: one of each size, centered along the rail)
- Users install standard knockout seals, cord grips, or conduit fittings after assembly
- For shared posts between bays, optional pass-through holes on the inward-facing tube walls allow cable routing between adjacent bays without exiting the frame

---

## 4. Input Configuration

### 4.1 TOML Config Structure

```toml
[machine]
name = "epoxy_coating_line"
description = "BLDC Stator Epoxy Powder Coating Machine"
units = "mm"                     # "mm" | "inch" — all dimensions in this unit

[frame]
# Tube selection — reference catalog by "size_wall" string
post_tube = "2x2_0.188"         # 2" × 2" A500, 0.188" wall
rail_tube = "2x2_0.120"         # 2" × 2" A500, 0.120" wall (lighter for rails)
cross_member_tube = "1.5x1.5_0.083"  # for internal cross-bracing / shelf rails

# Joint style
joint_type = "cope_with_tab"    # "cope" | "cope_with_tab" | "miter"
tab_width_in = 0.75             # tab width in inches
slot_clearance_in = 0.005       # per-side clearance for tube slots

# Topology
diagonal_bracing = false        # X-bracing on back face
top_cross_members = 3           # number of cross-members spanning the top frame
mid_shelf_rail = true           # add a horizontal rail at shelf height

[bay_defaults]
interior_width = 980            # mm — interior clear width per bay
interior_depth = 600            # mm — interior clear depth
interior_height = 850           # mm — interior clear height
shelf_height = 425              # mm from bottom interior

[bays]
# Define bay sequence left-to-right. Omit to auto-compute from overall_length.
count = 5

[[bays.overrides]]
index = 0
label = "Control Panel"
interior_width = 980

[[bays.overrides]]
index = 2
label = "Cyclone Separator"
interior_width = 980
shelf_height = 400

[panels]
side_gauge = 16                 # ga — side/back panels (non-structural skin)
front_gauge = 16                # ga — front panels / door panels
shelf_gauge = 14                # ga — shelves (higher load)
bottom_gauge = 14               # ga — bottom panel
panel_attachment = "slot_in_tube"  # "slot_in_tube" | "bolt_through"

# Front face config
front_type = "hinged_doors"     # "hinged_doors" | "removable_panels" | "open"
door_hinge = "piano"            # "piano" | "butt_2x" | "butt_3x"
door_latch = "quarter_turn"     # "quarter_turn" | "push_to_close" | "none"

# Back face config
back_type = "vented"            # "solid" | "vented" | "open"
vent_pattern = "round_perf"     # "round_perf" | "louver_slots" | "hex_perf"
vent_open_area = 0.40           # 40% open area target

[conveyor]
enabled = true
side = "front"
mount_type = "welded_plate"     # "oversized_rail" | "welded_plate" | "tslot_adapter"
plate_height = 200              # mm — mounting plate height above frame top
bolt_spacing = 100              # mm — M8 hole spacing along plate length
belt_width = 200                # mm — for documentation / clearance checking

[feet]
# Foot plates are laser-cut flat plates welded to the bottom of each post.
# Configurable bolt pattern for leveling feet, casters, or combo units.
style = "leveling_foot"         # "leveling_foot" | "caster" | "combo_leveling_caster"
plate_size = [120, 120]         # mm — square foot plate (laser cut, same gauge as structural)
plate_gauge = 11                # ga — 3.0mm, welded to post bottom
thread = "M16"                  # for leveling foot: M12 | M16 | M20 | "5/8-11" | "3/4-10"
adjustment_range = 25           # mm

# Caster options (when style = "caster" or "combo_leveling_caster")
caster_bolt_pattern = [76, 76]  # mm — bolt hole spacing on caster top plate
caster_bolt_diameter = 10       # mm — clearance hole diameter
caster_capacity_kg = 500        # per-caster rating (for BOM recommendation)

[worktop]
# Exports a plain DXF worktop outline sized to the top frame.
# User customizes for cutouts, material choice (steel vs wood).
enabled = true
material = "reference_only"     # exports outline only — user picks material
overhang = [0, 0, 0, 0]        # mm [front, back, left, right] beyond frame
cross_member_locations = true   # include cross-member positions as reference lines
mounting_holes = true           # pre-locate holes aligned with top frame members

[cable_routing]
# Standard electrical knockout holes in bottom rails / shared posts
enabled = true
knockout_sizes = ["1/2", "3/4", "1"]  # trade sizes to include
# Actual hole diameters (full laser cutout, no holding tabs):
#   1/2" trade = 7/8" (22.2mm)
#   3/4" trade = 1-1/8" (28.6mm)
#   1" trade   = 1-3/8" (34.9mm)
placement = "bottom_rail"       # "bottom_rail" | "shared_post" | "both"
holes_per_bay = 3               # one of each knockout size per bay

[powder_coat]
# All clearances adjusted for coating buildup.
# Typical powder coat: 2-4 mil (0.05-0.10mm) per surface.
enabled = true
thickness_per_surface_mm = 0.08 # mm per surface (conservative)
# Adds 2×thickness to slot widths, subtracts 2×thickness from tab widths.

[output]
formats = ["step", "dxf", "bom_csv", "bom_json", "assembly_pdf"]
dxf_units = "mm"
step_units = "mm"
include_nesting = true          # generate nested DXF for flat parts
sheet_size = [1220, 2440]       # mm — standard 4'×8' sheet for nesting
```

### 4.2 Tube Selection Helper

If the user doesn't know which tube to pick, a selection mode:

```bash
steelcab select-tube \
  --total-weight 3000 \        # kg total machine weight
  --bay-count 5 \
  --height 850 \
  --safety-factor 2.0

# Output:
# Recommended post tube: 2x2_0.188 (A500)
#   - Column buckling load per post: 18,400 lbs
#   - Load per post at SF=2.0: 3,307 lbs (1,500 kg)
#   - 8 shared posts × 1,500 kg = 12,000 kg capacity (OK for 3,000 kg)
# Recommended rail tube: 2x2_0.120 (A500)
#   - Mid-span deflection under 200kg shelf load: 0.3mm (L/3200)
```

---

## 5. Output Artifacts

### 5.1 STEP Files (Tube Parts)

One STEP file per unique tube member. OSH Cut accepts STEP uploads directly with their tube quoting engine.

```
output/epoxy_coating_line/
├── tubes/
│   ├── post_corner_FL.step          # Front-Left corner post
│   ├── post_corner_FR.step          # Front-Right corner post
│   ├── post_shared_F_01.step        # Front shared post between bay 1-2
│   ├── post_shared_F_02.step        # ... etc
│   ├── post_shared_B_01.step
│   ├── post_corner_BL.step
│   ├── post_corner_BR.step
│   ├── rail_top_front_01.step       # Top front rail, bay 1
│   ├── rail_top_front_02.step
│   ├── rail_top_back_01.step
│   ├── rail_btm_front_01.step
│   ├── rail_top_left_end.step       # End cap rail (left side of machine)
│   ├── rail_top_right_end.step
│   ├── cross_member_top_01.step     # Top cross-member
│   ├── cross_member_shelf_01.step   # Shelf-height cross rail
│   └── brace_back_01.step          # Diagonal brace (if enabled)
```

Each STEP file is a solid body representing the tube profile extruded to length, with all features:
- Cope profiles at ends
- Tab protrusions at cope joints
- Slots for flat panel insertion (cut through one wall)
- Bolt holes for removable panels
- Etched part ID text (laser engraving — cut depth 0.002")

### 5.2 DXF Files (Flat Parts)

One DXF per unique flat panel, plus nested layouts.

```
output/epoxy_coating_line/
├── panels/
│   ├── side_left.dxf               # Left end panel
│   ├── side_right.dxf              # Right end panel
│   ├── back_bay01.dxf              # Back panel for bay 1
│   ├── back_bay02.dxf
│   ├── door_bay01.dxf              # Front door for bay 1
│   ├── door_bay02.dxf
│   ├── bottom_bay01.dxf            # Bottom panel for bay 1
│   ├── shelf_bay02.dxf             # Shelf panel for bay 2
│   ├── conveyor_mount_plate.dxf    # Conveyor mounting plate
│   ├── gusset_shelf_L.dxf          # Shelf support gusset (L-bracket before bend)
│   └── gusset_shelf_R.dxf
├── foot_plates/
│   ├── foot_plate_leveling.dxf     # Foot plate with center weld-nut hole
│   ├── foot_plate_caster.dxf       # Foot plate with 4-hole caster bolt pattern
│   └── foot_plate_combo.dxf        # Foot plate with combo leveling caster pattern
├── worktop/
│   └── worktop_outline.dxf         # Plain worktop DXF with cross-member refs
├── nesting/
│   ├── nest_16ga_sheet01.dxf       # Nested layout on 4'×8' sheet
│   ├── nest_16ga_sheet02.dxf
│   └── nest_14ga_sheet01.dxf
```

**DXF layer conventions (same as v0.1):**

| Layer | Color | Content |
|-------|-------|---------|
| CUT | Red (1) | Through-cut lines |
| BEND | Blue (5) | Bend lines (reference) |
| ENGRAVE | Green (3) | Part ID, orientation marks |
| DIMS | Cyan (4) | Reference dimensions |

### 5.3 BOM (CSV + JSON)

```csv
part_id,type,material,profile,thickness_in,qty,length_mm,width_mm,weight_kg,source,unit_cost_est
POST-FL,tube,A500,2x2_0.188,0.188,1,850,,3.1,oshcut,
POST-FR,tube,A500,2x2_0.188,0.188,1,850,,3.1,oshcut,
POST-S-F01,tube,A500,2x2_0.188,0.188,1,850,,3.1,oshcut,
RAIL-TF-01,tube,A500,2x2_0.120,0.120,1,980,,2.2,oshcut,
PANEL-SIDE-L,sheet,A1008,flat,0.060,1,600,850,4.9,oshcut,
PANEL-BACK-01,sheet,A1008,flat,0.060,1,980,850,6.4,oshcut,
DOOR-01,sheet,A1008,flat,0.060,1,490,800,3.0,oshcut,
SHELF-01,sheet,A1008,flat,0.075,1,976,596,7.0,oshcut,
GUSSET-SHELF-L,sheet,A1008,flat,0.075,4,100,100,0.12,oshcut,
FOOT-PLATE-01,sheet,A1008,flat,0.120,8,120,120,0.11,oshcut,
WORKTOP-REF,sheet,reference,-,-,1,4900,2500,-,-,
CONV-PLATE-01,sheet,A1008,flat,0.120,1,980,200,1.9,oshcut,
FOOT-M16-LVL,hardware,,,,,8,,,mcmaster,8.50
KNOCKOUT-SEAL-0.5,hardware,,,,,5,,,mcmaster,1.50
KNOCKOUT-SEAL-0.75,hardware,,,,,5,,,mcmaster,1.75
KNOCKOUT-SEAL-1.0,hardware,,,,,5,,,mcmaster,2.00
BOLT-M6x16-BTN,hardware,,,,,48,,,mcmaster,0.35
HINGE-PIANO-800,hardware,,,,5,,,,,
LATCH-QTR-TURN,hardware,,,,5,,,,,
```

### 5.4 Assembly Guide (PDF/SVG)

Auto-generated exploded view showing:
- Assembly sequence (numbered steps)
- Weld symbol locations
- Part ID labels matching BOM
- Conveyor rail mounting detail

---

## 6. Geometry Engine

### 6.1 STEP Generation for Tubes

Use **CadQuery** (build123d also acceptable) to generate tube STEP files:

```python
import cadquery as cq

def make_post(length_mm, tube_od_mm, wall_mm, slots, tabs):
    """Generate a vertical post with panel slots and joint features."""
    od = tube_od_mm
    inner = od - 2 * wall_mm
    corner_r = 2 * wall_mm  # conservative corner radius
    
    # Base tube extrusion
    outer = cq.Workplane("XY").rect(od, od).val()
    inner = cq.Workplane("XY").rect(inner, inner).val()
    profile = outer.cut(inner)  # hollow section
    post = cq.Workplane("XY").add(profile).extrude(length_mm)
    
    # Cut panel slots (through one wall only)
    for slot in slots:
        post = post.cut(make_slot(slot))
    
    # Cut joint tab-receiving slots
    for tab_slot in tabs:
        post = post.cut(make_tab_slot(tab_slot))
    
    return post
```

### 6.2 DXF Generation for Flat Panels

Use **ezdxf** for flat panel DXF output:

```python
import ezdxf

def make_panel_dxf(width_mm, height_mm, slots, holes, part_id):
    """Generate a DXF for a flat panel with edge tabs and mounting holes."""
    doc = ezdxf.new(dxfversion='R2010', units=4)  # units=4 → millimeters
    msp = doc.modelspace()
    
    # Outer perimeter (with tab protrusions if slot-in-tube attachment)
    # ... polyline generation with tab geometry
    
    # Vent pattern (if back panel with vents)
    # ... parametric perforation array
    
    # Part ID engrave
    msp.add_text(part_id, dxfattribs={'layer': 'ENGRAVE', 'height': 5})
    
    doc.saveas(f"{part_id}.dxf")
```

### 6.3 Cope Profile Generation

For coping a rail end to wrap around a post:

1. Project the post's cross-section profile onto the end of the rail
2. Subtract the post profile from the rail end to create the cope notch
3. Leave tab protrusions within the coped region

This is a 3D boolean operation — CadQuery handles it natively:

```python
def cope_rail_to_post(rail, post, joint_position):
    """Cope the rail end to wrap around the post at the given position."""
    # Translate post to joint position
    # Boolean subtract post from rail end
    # Add tab features (small rect extrusion on coped face)
    return coped_rail
```

### 6.4 Panel Edge Profile for Slot-in-Tube

When a flat panel slides into a tube slot, the panel edge needs to be dimensioned:

- Panel edge engagement depth: 0.25" (6.35mm) into the slot
- Panel overall dimension = interior clear span + (2 × engagement depth)
- Slot length in tube: panel edge length minus corner offsets
- Slot width in tube wall: sheet thickness + 0.010" (0.254mm)

For panels that span multiple bays (like a continuous back panel), the panel is segmented per bay with a small gap at shared posts, or split into per-bay panels.

---

## 7. Tube Stock Constraints & Validation

### 7.1 Length Limits

| Source | Max Tube Length | Notes |
|--------|----------------|-------|
| OSH Cut | 235" (5969mm) | Covers all frame members for both machines |
| SendCutSend | 60" (1524mm) | Only 2"×2" profiles; suitable for short members only |

The tool validates that no generated tube member exceeds the max length for its source. For the epoxy machine, the longest continuous member would be a top-front rail at ~980mm per bay — well within limits. The only member that could approach the limit would be a continuous top rail across all bays, but the design splits these per-bay at shared posts.

### 7.2 Feature Constraints (Tube Laser)

Per OSH Cut design rules:
- **No blind features**: all cuts must go completely through the tube wall
- **No features crossing corner radii**: slots and holes must stay on flat faces, offset from corners by at least the corner radius
- **Minimum feature size**: slot width ≥ 1× wall thickness (to avoid burnthrough)
- **Bevel angles**: up to 45° (for cope joints)
- **Tab protrusion**: tabs must be continuous with the tube wall (no floating features)
- **Weld seam awareness**: critical features should avoid the tube weld seam location

The geometry engine validates all features against these rules and flags violations.

### 7.3 Minimum Clearances

```python
TUBE_SLOT_CLEARANCE = 0.005  # inches per side (for tube-to-tube tab joints)
PANEL_SLOT_CLEARANCE = 0.005  # inches per side (for sheet-in-tube slots)
CORNER_RADIUS_FACTOR = 2.0    # multiplier × wall thickness for corner radius
FEATURE_OFFSET_FROM_CORNER = 1.5  # × corner_radius, min distance from tube corner
MIN_TAB_ASPECT = 1.5          # tab length / tab width minimum
```

---

## 8. CLI Interface

```bash
# Generate all outputs from config
steelcab generate config.toml -o output/

# Select appropriate tube sizes for a given load
steelcab select-tube --weight 3000 --bays 5 --height 850

# Validate config without generating
steelcab validate config.toml

# Generate a single bay (quick prototyping)
steelcab quick-bay \
  --width 600 --depth 600 --height 850 \
  --post-tube 1.5x1.5_0.083 \
  --rail-tube 1.5x1.5_0.065 \
  --panel-gauge 16 \
  -o output/

# Export 3D preview (requires CadQuery)
steelcab preview config.toml --format step  # full assembly STEP
steelcab preview config.toml --format stl   # for quick 3D viewer

# Cost estimate (hits OSH Cut pricing if available, otherwise uses weight-based estimate)
steelcab estimate config.toml
```

---

## 9. Technology Stack

| Component | Library | Notes |
|-----------|---------|-------|
| Core | Python 3.11+ | |
| 3D geometry (tube STEP) | CadQuery 2.x | Parametric solid modeling, STEP export |
| 2D geometry (flat DXF) | ezdxf | DXF R2010 output with layers |
| Config | tomllib (stdlib 3.11) | TOML parsing |
| BOM | csv + json (stdlib) | |
| Nesting (optional) | `rectpack` or custom | Bin-packing for flat parts on sheets |
| CLI | click | |
| Testing | pytest | Geometry validation, BOM totals |
| Assembly PDF | svgwrite or reportlab | Exploded views + weld maps |

---

## 10. Example Configs

### 10.1 Epoxy Powder Coating Machine

```toml
[machine]
name = "epoxy_coating_line_v1"
description = "BLDC Stator Epoxy Powder Coating Machine - 1500pcs/hr"
units = "mm"

[frame]
post_tube = "2x2_0.188"
rail_tube = "2x2_0.120"
cross_member_tube = "1.5x1.5_0.083"
joint_type = "cope_with_tab"
top_cross_members = 3
mid_shelf_rail = true
diagonal_bracing = false

[bay_defaults]
interior_width = 980
interior_depth = 600
interior_height = 850
shelf_height = 400

[bays]
count = 5

[[bays.overrides]]
index = 0
label = "Control Panel Bay"

[[bays.overrides]]
index = 2
label = "Cyclone / Powder Recovery"

[[bays.overrides]]
index = 4
label = "HF Power Supply Bay"

[panels]
side_gauge = 16
front_gauge = 16
shelf_gauge = 14
bottom_gauge = 14
panel_attachment = "slot_in_tube"
front_type = "hinged_doors"
back_type = "vented"
vent_pattern = "round_perf"
vent_open_area = 0.40

[conveyor]
enabled = true
side = "front"
mount_type = "welded_plate"
plate_height = 200
bolt_spacing = 100
belt_width = 200

[feet]
style = "leveling_foot"
plate_size = [120, 120]
plate_gauge = 11
thread = "M16"
adjustment_range = 25

[worktop]
enabled = true
material = "reference_only"
overhang = [25, 0, 25, 25]     # 25mm overhang front/left/right for edge access

[cable_routing]
enabled = true
knockout_sizes = ["1/2", "3/4", "1"]
placement = "both"
holes_per_bay = 3

[powder_coat]
enabled = true
thickness_per_surface_mm = 0.08
```

### 10.2 Stator Winding Machine

```toml
[machine]
name = "winding_machine_4sta"
description = "4-Station BLDC Stator Needle Winding Machine"
units = "mm"

[frame]
post_tube = "1.5x1.5_0.120"
rail_tube = "1.5x1.5_0.083"
cross_member_tube = "1x1_0.083"
joint_type = "cope_with_tab"
top_cross_members = 2
mid_shelf_rail = true

[bay_defaults]
interior_width = 580
interior_depth = 780
interior_height = 850
shelf_height = 400

[bays]
count = 2

[panels]
side_gauge = 16
front_gauge = 16
shelf_gauge = 14
bottom_gauge = 14
panel_attachment = "slot_in_tube"
front_type = "hinged_doors"
back_type = "vented"

[conveyor]
enabled = true
side = "front"
mount_type = "tslot_adapter"
belt_width = 150

[feet]
style = "combo_leveling_caster"
plate_size = [100, 100]
plate_gauge = 11
caster_bolt_pattern = [64, 64]
caster_bolt_diameter = 8
caster_capacity_kg = 300

[worktop]
enabled = true
material = "reference_only"
overhang = [0, 0, 0, 0]

[cable_routing]
enabled = true
knockout_sizes = ["1/2", "3/4"]
placement = "bottom_rail"
holes_per_bay = 2

[powder_coat]
enabled = true
thickness_per_surface_mm = 0.08
```

---

## 11. Resolved Design Decisions

| Decision | Resolution |
|----------|-----------|
| Worktop | Export a plain DXF outline of the worktop, sized to the top frame. Cross-member positions included as reference lines. User customizes for cutouts; material is user's choice (steel sheet or wood). |
| Powder coat | Assume everything gets powder coated or painted. All slot/tab clearances are automatically adjusted by `2 × powder_coat_thickness_per_surface` (default 0.08mm/surface = 0.16mm added to each slot, subtracted from each tab). |
| Output format for tubes | 3D STEP files for tube parts (OSH Cut accepts STEP directly). |
| Cable pass-through | Standard NEC knockout holes laser-cut in bottom rails. Full cutouts (no holding tabs). Sizes: 1/2" trade (22.2mm), 3/4" trade (28.6mm), 1" trade (34.9mm). |
| Foot mounting | Laser-cut foot plates welded to the bottom of each post. Configurable bolt pattern for leveling feet, plate-mount casters, or combination leveling casters. |

## 12. Open Questions

1. **Post continuity at shared bay boundaries** — Current design makes posts continuous floor-to-top, with rails coping into them. For shipping, should posts be splittable (two half-posts with a sleeve joint at mid-height)?

2. **Panel edge treatment** — Should panels that slot into tube have small tabs at intervals along the edge for tack-welding, or continuous straight edges that rely on plug welds through the tube slot?

3. **Inter-bay alignment pins** — For the bolted-splice configuration, should the tool generate alignment pin holes (e.g., 2× dowel pin holes per shared post pair) for precise bay-to-bay registration?

4. **Foot plate weld prep** — Should the foot plate DXF include a bevel or chamfer on the inner edge (matching the tube OD) for better weld penetration, or keep it flat and let the welder prep?

5. **Conveyor side clearance** — The conveyor mounting plate extends above the cabinet top. Should the tool generate clearance notches in the worktop DXF where the mounting plate protrudes?

6. **Knockout placement logic** — Should knockouts be centered on each bottom rail, or biased toward the back of the cabinet (closer to where cables typically route)?
