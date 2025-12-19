# Task_Floorplan_ICC2 – GitHub Documentation

This repository contains an **ICC2 floorplan-only** setup for the `vsdcaravel` SoC design using Synopsys ICC2 2022.12. 

***

## Project overview

This project implements Task 5 of the ICC2 SoC floorplanning workshop for the `vsdcaravel` design using the SCL 180 nm PDK. The flow stops strictly at **floorplan creation**, with no placement, CTS, routing, or PDN steps enabled. 

Key goals:

- Create a die with exact target dimensions.
- Define core margins inside the die.
- Reserve IO regions along all four sides of the die.
- Visualize and report the final floorplan and top-level ports.

***

- `scripts/floorplan.tcl`: Main ICC2 TCL script that creates the design library and floorplan.
- `reports/floorplan_report.txt`: Text report capturing die/core geometry and top-level ports.

<img width="1604" height="1017" alt="image" src="https://github.com/user-attachments/assets/5f3a5117-20b9-464a-8885-e72a97cd1e15" />

***

## Floorplan configuration

### Die size

The die is initialized using die-controlled floorplanning with the following dimensions (in microns):  

- **Die width**: 3.588 mm → 3588 µm. 
- **Die height**: 5.188 mm → 5188 µm. 

In the script:

```tcl
initialize_floorplan \
    -control_type die \
    -boundary {{0 0} {3588 5188}} \
    -core_offset {200 200 200 200}
```

This sets the die boundary from \((0,0)\) to \((3588,5188)\) in the ICC2 coordinate system. 

### Core margin

A uniform **core margin of 200 µm on all four sides** is used via `-core_offset {200 200 200 200}`. This results in a core area of: 

- Core lower-left: \((200, 200)\)  
- Core upper-right: \((3388, 4988)\)  

These values are also echoed in `floorplan_report.txt`:

```tcl
puts "Die Area  : 0 0 3588 5188  (microns)"
puts "Core Area : 200 200 3388 4988  (microns)"
```

***

## IO placement strategy

For this task, **IO regions are created using hard placement blockages** along all four edges of the die. These blockages reserve space along the boundary where IO pads/ports can be placed and prevent standard-cell placement there. 

### IO regions using placement blockages

The script defines four IO regions as hard blockages:

```tcl
# Bottom IO region
create_placement_blockage \
  -name IO_BOTTOM \
  -type hard \
  -boundary {{0 0} {3588 100}}

# Top IO region
create_placement_blockage \
  -name IO_TOP \
  -type hard \
  -boundary {{0 5088} {3588 5188}}

# Left IO region
create_placement_blockage \
  -name IO_LEFT \
  -type hard \
  -boundary {{0 100} {100 5088}}

# Right IO region
create_placement_blockage \
  -name IO_RIGHT \
  -type hard \
  -boundary {{3488 100} {3588 5088}}
```

- Bottom strip: 100 µm tall along the entire die width.
- Top strip: 100 µm tall along the entire die width.
- Left strip: 100 µm wide along most of die height.
- Right strip: 100 µm wide along most of die height.

These regions provide **continuous IO bands** around the die, enabling even distribution of IO ports and avoiding overlap with core placement. 

### Port visualization and placement

After reading the synthesized netlist, top-level ports are queried and viewed using:

<img width="1606" height="1021" alt="image" src="https://github.com/user-attachments/assets/c63188a9-1135-46c0-abf5-b10112699dc6" />

<img width="1604" height="1018" alt="image" src="https://github.com/user-attachments/assets/0da72bee-0a37-4cea-909a-df312c9d9fc1" />

```tcl
gui_show_man_page
```
<img width="1614" height="897" alt="Screenshot 2025-12-19 191716" src="https://github.com/user-attachments/assets/b978b905-9a15-4ef3-baa3-a308c488f1a5" />

```tcl
get_ports
```

This command is used both in the console and within the redirected report section to list all top-level ports. 

Port placement is performed with:

<img width="1612" height="308" alt="Screenshot 2025-12-19 191923" src="https://github.com/user-attachments/assets/aa530458-8316-4115-9ce6-e4f1275844cd" />

<img width="1619" height="937" alt="Screenshot 2025-12-19 191948" src="https://github.com/user-attachments/assets/a72fb917-9126-4a2d-8d4c-8891ff277c09" />

```tcl
place_ports -self
```

- `place_ports -self` automatically places top-level ports around the die boundary, constrained by the defined floorplan and IO regions.
- The result shows ports aligned along the four edges, respecting IO region constraints.

***

## Script details (`scripts/floorplan.tcl`)

The `floorplan.tcl` script automates creation of the floorplan-only design context in ICC2.

```tcl
############################################################
# Task 5 – SoC Floorplanning Using ICC2 (Floorplan Only)
# Design  : vsdcaravel
# Tool    : Synopsys ICC2 2022.12
############################################################

# ---------------------------------------------------------
# Basic Setup
# ---------------------------------------------------------
set DESIGN_NAME      vsdcaravel
set DESIGN_LIBRARY   vsdcaravel_fp_lib

# ---------------------------------------------------------
# Reference Library (includes technology internally)
# ---------------------------------------------------------
# Using unified NDM library provided with ICC2 workshop
set REF_LIB \
"/home/Synopsys/pdk/SCL_PDK_3/work/run1/icc2_workshop_collaterals/standaloneFlow/work/raven_wrapperNangate/lib.ndm"

# ---------------------------------------------------------
# Create ICC2 Design Library
# ---------------------------------------------------------
if {[file exists $DESIGN_LIBRARY]} {
    file delete -force $DESIGN_LIBRARY
}

create_lib $DESIGN_LIBRARY \
    -ref_libs $REF_LIB

# ---------------------------------------------------------
# Read Synthesized Netlist
# (Netlist is read only to create design context;
# unresolved cells are acceptable for floorplan-only task)
# ---------------------------------------------------------
read_verilog -top $DESIGN_NAME "/home/prakhan/Downloads/task4/vsdRiscvScl180/synthesis/output/vsdcaravel_synthesis.v"

current_design $DESIGN_NAME

# ---------------------------------------------------------
# Floorplan Definition (MANDATORY)
# Die Size  : 3.588 mm × 5.188 mm
# Core Margin : 200 µm on all sides
#
# NOTE:
# This ICC2 version requires die-controlled initialization
# using -control_type die and -boundary syntax.
# ---------------------------------------------------------
initialize_floorplan \
    -control_type die \
    -boundary {{0 0} {3588 5188}} \
    -core_offset {200 200 200 200}
# ---------------------------------------------------------
# IO Regions using Placement Blockages (Corrected)
# ---------------------------------------------------------

# Bottom IO region (along bottom die edge)
create_placement_blockage \
  -name IO_BOTTOM \
  -type hard \
  -boundary {{0 0} {3588 100}}

# Top IO region (along top die edge)
create_placement_blockage \
  -name IO_TOP \
  -type hard \
  -boundary {{0 5088} {3588 5188}}

# Left IO region (along left die edge)
create_placement_blockage \
  -name IO_LEFT \
  -type hard \
  -boundary {{0 100} {100 5088}}

# Right IO region (along right die edge)
create_placement_blockage \
  -name IO_RIGHT \
  -type hard \
  -boundary {{3488 100} {3588 5088}}


# ---------------------------------------------------------
# Macro Placement
# ---------------------------------------------------------
# NOTE:
# No physical hard macros exist in this design.
# RAM128 and RAM256 are RTL-based memory models that were
# synthesized into logic and optimized away.
# Therefore, no macro placement is performed here.
# ---------------------------------------------------------

# ---------------------------------------------------------
# Reports
# ---------------------------------------------------------
redirect -file ../reports/floorplan_report.txt {

    puts "===== FLOORPLAN GEOMETRY (USER DEFINED) ====="
    puts "Die Area  : 0 0 3588 5188  (microns)"
    puts "Core Area : 200 200 3388 4988  (microns)"

    puts "\n===== TOP LEVEL PORTS ====="
    get_ports
}

```

### 1. Basic design and library setup

```tcl
set DESIGN_NAME      vsdcaravel
set DESIGN_LIBRARY   vsdcaravel_fp_lib
```

- `DESIGN_NAME`: Name of the top-level SoC RTL/netlist (`vsdcaravel`).
- `DESIGN_LIBRARY`: Name of the ICC2 design library created for floorplanning.

### 2. Reference NDM library

```tcl
set REF_LIB \
"/home/Synopsys/pdk/SCL_PDK_3/work/run1/icc2_workshop_collaterals/standaloneFlow/work/raven_wrapperNangate/lib.ndm"
```

This unified NDM library contains the reference technology and standard cells used during floorplanning for this workshop environment. 

### 3. ICC2 design library creation

```tcl
if {[file exists $DESIGN_LIBRARY]} {
    file delete -force $DESIGN_LIBRARY
}

create_lib $DESIGN_LIBRARY \
    -ref_libs $REF_LIB
```

- Any existing library with the same name is deleted to ensure a clean run.
- `create_lib` creates a new ICC2 library and attaches the reference NDM file.

### 4. Read synthesized netlist and set top

```tcl
read_verilog -top $DESIGN_NAME \
  "/home/prakhan/Downloads/task4/vsdRiscvScl180/synthesis/output/vsdcaravel_synthesis.v"

current_design $DESIGN_NAME
```

- Reads the synthesized Verilog of `vsdcaravel` as the top design.
- Unresolved cells are acceptable since the flow stops after floorplanning.

### 5. Floorplan creation

The floorplan is created using `initialize_floorplan` with explicit die boundary and core margin, as detailed earlier.

### 6. Macro placement

The script documents that no hard macros exist:

```tcl
# No physical hard macros exist in this design.
# RAM128 and RAM256 are RTL-based memory models that were
# synthesized into logic and optimized away.
# Therefore, no macro placement is performed here.
```

This clarifies that macro placement is intentionally skipped because memories are implemented as standard-cell logic, not hard macro instances.

### 7. Reporting

At the end of the script, floorplan and port information is written to `../reports/floorplan_report.txt` using `redirect`:

```tcl
redirect -file ../reports/floorplan_report.txt {

    puts "===== FLOORPLAN GEOMETRY (USER DEFINED) ====="
    puts "Die Area  : 0 0 3588 5188  (microns)"
    puts "Core Area : 200 200 3388 4988  (microns)"

    puts "\n===== TOP LEVEL PORTS ====="
    get_ports
}
```

- Logs die and core geometry.
- Lists all top-level ports for documentation and verification.


***

## Changes from reference script

The flow is adapted from the ICC2 workshop standalone flow, but customized for `vsdcaravel` and the specific floorplan task. 

Main modifications:

- **Design-specific configuration**
  - Changed design name to `vsdcaravel`.
  - Updated netlist path to the synthesized `vsdcaravel_synthesis.v`.

- **Target die dimensions**
  - Set die boundary to \((0,0)\) – \((3588,5188)\) to match 3.588 mm × 5.188 mm requirement. 

- **Core margin**
  - Introduced uniform 200 µm core offset on all sides via `-core_offset`.

- **IO strategy**
  - Replaced or added IO-related commands with `create_placement_blockage` definitions for `IO_TOP`, `IO_BOTTOM`, `IO_LEFT`, and `IO_RIGHT` to explicitly carve out IO bands. 

- **Flow restriction to floorplan-only**
  - Omitted all placement, CTS, routing, and PDN-related commands.
  - Stopped the flow after floorplan creation and reporting, as required by the task. 

***

## How to run

From within an ICC2 shell:

1. Set up your ICC2 environment and technology paths (consistent with the workshop setup).
2. Create a working directory and link this repository.
3. Run:

   ```tcl
   source scripts/floorplan.tcl
   ```

4. After execution:
   - Open the floorplan in the ICC2 GUI for visualization.
   - Verify die/core boundaries and IO regions.
   - Optionally run `place_ports -self` in the ICC2 shell to auto-place ports and update the GUI view.

---

## Verification checklist

Use this checklist to confirm the task requirements are met. 

- Die size exactly \(3588 µm × 5188 µm\).
- Core margins of 200 µm on all four sides.
- IO regions present along all four edges (top, bottom, left, right).
- `get_ports` lists all top-level ports in `floorplan_report.txt`.
- Ports are visible around the boundary after `place_ports -self`.
- Screenshots in `images/` clearly show:
  - Die boundary.
  - Core boundary.
  - IO regions and ports near the edges.
