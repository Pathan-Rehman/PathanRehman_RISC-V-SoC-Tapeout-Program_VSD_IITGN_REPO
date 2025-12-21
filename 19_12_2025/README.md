# Task_Floorplan_ICC2 – GitHub Documentation

This repository contains an **ICC2 floorplan setup** for the design using Synopsys ICC2. The flow implements floorplan initialization with proper library setup, technology configuration, and design geometry definition.

***

## Project Overview

Reffered Repository: https://github.com/kunalg123/icc2_workshop_collaterals

Make the path changes and run the scripts.

This project implements ICC2 SoC floorplanning using a modular setup approach. The flow creates a design library, reads synthesized netlists, configures technology parameters, and initializes the floorplan with precise die and core dimensions.

Key goals:

- Create a design library with proper technology and reference library setup
- Read synthesized Verilog netlist with support for hierarchical flows
- Configure technology setup including routing layers and parasitic extraction
- Initialize floorplan with exact die dimensions and core margins
- Save the floorplan for subsequent design stages

<img width="1608" height="1018" alt="image" src="https://github.com/user-attachments/assets/e5afe3c2-e351-473b-bc26-761eab46bf97" />


***

## Repository Structure

- `icc2_common_setup.tcl`: Common ICC2 setup variables and configurations
- `icc2_dp_setup.tcl`: Design planning specific setup variables
- `reports/`: Directory containing design check and floorplan reports
- Design library created in `${WORK_DIR}/${DESIGN_LIBRARY}`

***

```floorplan.tcl
source -echo ./icc2_common_setup.tcl
source -echo ./icc2_dp_setup.tcl
if {[file exists ${WORK_DIR}/$DESIGN_LIBRARY]} {
   file delete -force ${WORK_DIR}/${DESIGN_LIBRARY}
}
###---NDM Library creation---###
set create_lib_cmd "create_lib ${WORK_DIR}/$DESIGN_LIBRARY"
if {[file exists [which $TECH_FILE]]} {
   lappend create_lib_cmd -tech $TECH_FILE ;# recommended
} elseif {$TECH_LIB != ""} {
   lappend create_lib_cmd -use_technology_lib $TECH_LIB ;# optional
}
lappend create_lib_cmd -ref_libs $REFERENCE_LIBRARY
puts "RM-info : $create_lib_cmd"
eval ${create_lib_cmd}

###---Read Synthesized Verilog---###
if {$DP_FLOW == "hier" && $BOTTOM_BLOCK_VIEW == "abstract"} {
   # Read in the DESIGN_NAME outline.  This will create the outline
   puts "RM-info : Reading verilog outline (${VERILOG_NETLIST_FILES})"
   read_verilog_outline -design ${DESIGN_NAME}/${INIT_DP_LABEL_NAME} -top ${DESIGN_NAME} ${VERILOG_NETLIST_FILES}
   } else {
   # Read in the full DESIGN_NAME.  This will create the DESIGN_NAME view in the database
   puts "RM-info : Reading full chip verilog (${VERILOG_NETLIST_FILES})"
   read_verilog -design ${DESIGN_NAME}/${INIT_DP_LABEL_NAME} -top ${DESIGN_NAME} ${VERILOG_NETLIST_FILES}
}

## Technology setup for routing layer direction, offset, site default, and site symmetry.
#  If TECH_FILE is specified, they should be properly set.
#  If TECH_LIB is used and it does not contain such information, then they should be set here as well.
if {$TECH_FILE != "" || ($TECH_LIB != "" && !$TECH_LIB_INCLUDES_TECH_SETUP_INFO)} {
   if {[file exists [which $TCL_TECH_SETUP_FILE]]} {
      puts "RM-info : Sourcing [which $TCL_TECH_SETUP_FILE]"
      source -echo $TCL_TECH_SETUP_FILE
   } elseif {$TCL_TECH_SETUP_FILE != ""} {
      puts "RM-error : TCL_TECH_SETUP_FILE($TCL_TECH_SETUP_FILE) is invalid. Please correct it."
   }
}

# Specify a Tcl script to read in your TLU+ files by using the read_parasitic_tech command
if {[file exists [which $TCL_PARASITIC_SETUP_FILE]]} {
   puts "RM-info : Sourcing [which $TCL_PARASITIC_SETUP_FILE]"
   source -echo $TCL_PARASITIC_SETUP_FILE
} elseif {$TCL_PARASITIC_SETUP_FILE != ""} {
   puts "RM-error : TCL_PARASITIC_SETUP_FILE($TCL_PARASITIC_SETUP_FILE) is invalid. Please correct it."
} else {
   puts "RM-info : No TLU plus files sourced, Parastic library containing TLU+ must be included in library reference list"
}

###---Routing settings---###
## Set max routing layer
if {$MAX_ROUTING_LAYER != ""} {set_ignored_layers -max_routing_layer $MAX_ROUTING_LAYER}
## Set min routing layer
if {$MIN_ROUTING_LAYER != ""} {set_ignored_layers -min_routing_layer $MIN_ROUTING_LAYER}

####################################
# Check Design: Pre-Floorplanning
####################################
if {$CHECK_DESIGN} {
   redirect -file ${REPORTS_DIR_INIT_DP}/check_design.pre_floorplan     {check_design -ems_database check_design.pre_floorplan.ems -checks dp_pre_floorplan}
}

####################################
# Floorplanning (USER-DEFINED)
####################################

initialize_floorplan \
    -control_type die \
    -boundary {{0 0} {3588 5188}} \
    -core_offset {200 200 200 200}

save_block -force -label floorplan
save_lib -all

```

## Floorplan Configuration

### Die Size

The die is initialized using die-controlled floorplanning with the following dimensions (in microns):

- **Die width**: 3.588 mm → 3588 µm
- **Die height**: 5.188 mm → 5188 µm

In the script:

```tcl
initialize_floorplan \
    -control_type die \
    -boundary {{0 0} {3588 5188}} \
    -core_offset {200 200 200 200}
```

This sets the die boundary from \((0,0)\) to \((3588,5188)\) in the ICC2 coordinate system.

### Core Margin

A uniform **core margin of 200 µm on all four sides** is used via `-core_offset {200 200 200 200}`. This results in a core area of:

- Core lower-left: \((200, 200)\)
- Core upper-right: \((3388, 4988)\)

***

## Script Details

The floorplan script implements a comprehensive setup and initialization flow.

### 1. Setup File Sourcing

```tcl
source -echo ./icc2_common_setup.tcl
source -echo ./icc2_dp_setup.tcl
```

<details>
  <summary>icc2_common_setup.tcl</summary>

	```puts "RM-info : Running script [info script]\n"

		##########################################################################################
		# Tool: IC Compiler II
# Script: icc2_common_setup.tcl
# Version: P-2019.03-SP4
# Copyright (C) 2014-2019 Synopsys, Inc. All rights reserved.
##########################################################################################

##########################################################################################
## Required variables
## These variables must be correctly filled in for the flow to run properly
##########################################################################################
set DESIGN_NAME 		"raven_wrapper" ;# Required; name of the design to be worked on; also used as the block name when scripts save or copy a block
set LIBRARY_SUFFIX		"Nangate" ;# Suffix for the design library name ; default is unspecified   
set DESIGN_LIBRARY 		"${DESIGN_NAME}${LIBRARY_SUFFIX}" ;# Name of the design library; default is ${DESIGN_NAME}${LIBRARY_SUFFIX}
set REFERENCE_LIBRARY 		[list /home/prakhan/task_5/icc2_workshop_collaterals/nangate_stdcell.lef /home/prakhan/task_5/icc2_workshop_collaterals/sram/sram_32_1024_freepdk45.lef]	;# Required; a list of reference libraries for the design library.
					;#	for library configuration flow (LIBRARY_CONFIGURATION_FLOW set to true below): 
					;#		- specify the list of physical source files to be used for library configuration during create_lib
				       	;# 	for hierarchical designs using bottom-up flows: include subblock design libraries in the list;
					;# 	for hierarchical designs using ETMs: include the ETM library in the list.
					;# 		- If unpack_rm_dirs.pl is used to create dir structures for hierarchical designs, 
					;#		  in order to transition between hierarchical DP and hierarchical PNR flows properly, 
					;#		  absolute paths are a requirement.
set COMPRESS_LIBS               "false" ;# Save libs as compressed NDM; only used in DP.
#set VERILOG_NETLIST_FILES      "/home/kunal/workshop/icc2_workshop_collaterals/pnrScripts/spi_slave.synth.v"
set VERILOG_NETLIST_FILES	"/home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v"	;# Verilog netlist files;
					;# 	for DP: required
					;# 	for PNR: required if INIT_DESIGN_INPUT is ASCII in icc2_pnr_setup.tcl; not required for DC_ASCII or DP_RM_NDM
set UPF_FILE 			""	;# A UPF file
					;# 	for DP: required
					;# 	for PNR: required if INIT_DESIGN_INPUT is ASCII in icc2_pnr_setup.tcl; not required for DC_ASCII or DP_RM_NDM
                                        ;#          for hierarchical designs using ETMs, load the block upf file
                                        ;#          for each sub-block linked to ETM, include the following line in the UPF_FILE 
                			;#              load_upf block.upf -scope block_instance_name
set UPF_SUPPLEMENTAL_FILE	""      ;# The supplemental UPF file. Only needed if you are running golden UPF flow, in which case, you need both UPF_FILE and this.
					;# 	for DP: required
					;# 	for PNR: required if INIT_DESIGN_INPUT is ASCII in icc2_pnr_setup.tcl; not required for DC_ASCII or DP_RM_NDM
					;#	    If UPF_SUPPLEMENTAL_FILE is specified, scripts assume golden UPF flow. load_upf and save_upf commands will be different.	

set TCL_PARASITIC_SETUP_FILE	"./init_design.read_parasitic_tech_example.tcl"	;# Specify a Tcl script to read in your TLU+ files by using the read_parasitic_tech command;
					;# refer to the example in templates/init_design.read_parasitic_tech_example.tcl 

#set TCL_MCMM_SETUP_FILE         ""
set TCL_MCMM_SETUP_FILE		"./init_design.mcmm_example.auto_expanded.tcl"	;# Specify a Tcl script to create your corners, modes, scenarios and load respective constraints;
					;# two examples are provided in templates/: 
					;# init_design.mcmm_example.explicit.tcl: provide mode, corner, and scenario constraints; create modes, corners, 
					;# and scenarios; source mode, corner, and scenario constraints, respectively 
					;# init_design.mcmm_example.auto_expanded.tcl: provide constraints for the scenarios; create modes, corners, 
					;# and scenarios; source scenario constraints which are then expanded to associated modes and corners
					;# 	for DP: required
					;# 	for PNR: required if INIT_DESIGN_INPUT is ASCII in icc2_pnr_setup.tcl; not required for DC_ASCII or DP_RM_NDM

set TECH_FILE 			"/home/prakhan/task_5/icc2_workshop_collaterals/nangate.tf" 	;# A technology file; TECH_FILE and TECH_LIB are mutually exclusive ways to specify technology information; 
					;# TECH_FILE is recommended, although TECH_LIB is also supported in ICC2 RM. 
set TECH_LIB			""	;# Specify the reference library to be used as a dedicated technology library;
                        		;# as a best practice, please list it as the first library in the REFERENCE_LIBRARY list 
set TECH_LIB_INCLUDES_TECH_SETUP_INFO true 
					;# Indicate whether TECH_LIB contains technology setup information such as routing layer direction, offset, 
					;# site default, and site symmetry, etc. TECH_LIB may contain this information if loaded during library prep.
					;# true|false; this variable is associated with TECH_LIB. 
set TCL_TECH_SETUP_FILE		"./init_design.tech_setup.tcl"
					;# Specify a TCL script for setting routing layer direction, offset, site default, and site symmetry list, etc.
					;# init_design.tech_setup.tcl is the default. Use it as a template or provide your own script.
					;# This script will only get sourced if the following conditions are met: 
					;# (1) TECH_FILE is specified (2) TECH_LIB is specified && TECH_LIB_INCLUDES_TECH_SETUP_INFO is false 
set ROUTING_LAYER_DIRECTION_OFFSET_LIST "{metal1 horizontal} {metal2 vertical} {metal3 horizontal} {metal4 vertical} {metal5 horizontal} {metal6 vertical} {metal7 horizontal} {metal8 vertical} {metal9 horizontal} {metal10 vertical}" 
					;# Specify the routing layers as well as their direction and offset in a list of space delimited pairs;
					;# This variable should be defined for all metal routing layers in technology file;
					;# Syntax is "{metal_layer_1 direction offset} {metal_layer_2 direction offset} ...";
					;# It is required to at least specify metal layers and directions. Offsets are optional. 
					;# Example1 is with offsets specified: "{M1 vertical 0.2} {M2 horizontal 0.0} {M3 vertical 0.2}"
					;# Example2 is without offsets specified: "{M1 vertical} {M2 horizontal} {M3 vertical}"
##########################################################################################
## Optional variables
## Specify these variables if the corresponding functions are desired 
##########################################################################################
set DESIGN_LIBRARY_SCALE_FACTOR	""	;# Specify the length precision for the library. Length precision for the design
					;# library and its ref libraries must be identical. Tool default is 10000, which
					;# implies one unit is one Angstrom or 0.1nm.

set UPF_UPDATE_SUPPLY_SET_FILE	""	;# A UPF file to resolve UPF supply sets

#set DEF_FLOORPLAN_FILES		"/home/kunal/design/scripts/pnr/ICC2-RM_P-2019.03-SP4/write_data_dir/picorv32/picorv32.icc.floorplan/floorplan.def.gz"	;# DEF files which contain the floorplan information;
set DEF_FLOORPLAN_FILES                ""  ;# DEF files which contain the floorplan information;
					;# 	for DP: not required
					;# 	for PNR: required if INIT_DESIGN_INPUT = ASCII in icc2_pnr_setup.tcl and neither TCL_FLOORPLAN_FILE or 
					;#		 initialize_floorplan is used; DEF_FLOORPLAN_FILES and TCL_FLOORPLAN_FILE are mutually exclusive;
					;# 	         not required if INIT_DESIGN_INPUT = DC_ASCII or DP_RM_NDM

set DEF_SCAN_FILE		""	;# A scan DEF file for scan chain information;
					;# 	for PNR: not required if INIT_DESIGN_INPUT = DC_ASCII or DP_RM_NDM, as SCANDEF is expected to be loaded already

set DEF_SITE_NAME_PAIRS		{}	;# A list of site name pairs for read_def -convert; 
					;# specify site name pairs with from_site first followed by to_site;
					;# Example: set DEF_SITE_NAME_PAIRS {{from_site_1 to_site_1} {from_site_2 to_site_2}} 	
set SITE_DEFAULT		""	;# Specify the default site name if there are multiple site defs in the technology file;
					;# this is to be used by initialize_floorplan command; example: set SITE_DEFAULT "unit";
					;# this is applied in the init_design.tech_setup.tcl script 
set SITE_SYMMETRY_LIST	""		;# Specify a list of site def and its symmetry value;
					;# this is to be used by read_def or initialize_floorplan command to control the site symmetry;
					;# example: set SITE_SYMMETRY_LIST "{unit Y} {unit1 Y}"; this is applied in the init_design.tech_setup.tcl script 

set MIN_ROUTING_LAYER		"metal1"	;# Min routing layer name; normally should be specified; otherwise tool can use all metal layers
set MAX_ROUTING_LAYER		"metal10"	;# Max routing layer name; normally should be specified; otherwise tool can use all metal layers

set LIBRARY_CONFIGURATION_FLOW	false	;# Set it to true enables library configuration flow which calls the library manager under the hood to generate .nlibs, 
					;# save them to disk, and automatically link them to the design.
					;# Requires LINK_LIBRARY to be specified with .db files and REFERENCE_LIBRARY to be specified with physical
					;# source files for the library configuration flow. Also search_path (in icc2_pnr_setup.tcl) should include paths 
					;# to these .db and physical source files.

set LINK_LIBRARY		[list /home/prakhan/task_5/icc2_workshop_collaterals/nangate_typical.db /home/prakhan/task_5/icc2_workshop_collaterals/sram_32_1024_freepdk45_TT_1p0V_25C_lib.db]	;# Specify .db files;
					;# 	for running VC-LP (vc_lp.tcl) and Formality (fm.tcl): required
					;# 	for ICC-II without LIBRARY_CONFIGURATION_FLOW enabled: not required
					;#	for ICC-II with LIBRARY_CONFIGURATION_FLOW enabled: required; 
					;#      	- the .db files specified will be used for the library configuration under the hood during create_lib 

##########################################################################################
## Variables related to flow controls of flat PNR, hierarchical PNR and transition with DP
##########################################################################################
set DESIGN_STYLE		"hier"	;# Specify the design style; flat|hier; default is flat; 
					;# specify flat for a totally flat flow (flat PNR for short) and 
					;# specify hier for a hierarchical flow (hier PNR for short);
					;# 	for hier PNR: required and auto set if unpack_rm_dirs.pl is used; (see README.unpack_rm_dirs.txt for details)
					;# 	for flat PNR: this should set to flat (default)
					;#	for DP: not used 

set PHYSICAL_HIERARCHY_LEVEL	"" 	;# Specify the current level of hierarchy for the hierarchical PNR flow; top|intermediate|bottom;
					;# 	for hier PNR: required and auto set if unpack_rm_dirs.pl is used; (see README.unpack_rm_dirs.txt for details)
					;# 	for flat PNR and for DP: not used.
set RELEASE_DIR_DP		"write_data_dir_hier" 	;# Specify the release directory of DP RM; 
					;# this is where init_design.tcl of PNR flow gets DP RM released libraries;
					;# 	for hier PNR: required and auto set if unpack_rm_dirs.pl is used; (see README.unpack_rm_dirs.txt for details)
					;# 	for flat PNR: required if INIT_DESIGN_INPUT = DP_RM_NDM, as init_design.tcl needs to know where DP RM libraries are
					;#	for DP: not used 
set RELEASE_LABEL_NAME_DP 	"rave_wrapperNangate"	
					;# Specify the label name of the block in the DP RM released library;
					;# this is the label name which init_design.tcl of PNR flow will open. 
set RELEASE_DIR_PNR		"" 	;# Specify the release directory of PNR RM; 
					;# this is where the init_design.tcl of hierarchical PNR flow gets the sub-block libraries;	
					;# 	for hier PNR: required and auto set if unpack_rm_dirs.pl is used; (see README.unpack_rm_dirs.txt for details)
					;# 	for flat PNR and for DP: not used.
##########################################################################################
## Variables related to REDHAWK ANALYSIS FUSION
##########################################################################################
set REDHAWK_SEARCH_PATH		"" 	;# Required. Search path to the NDM, reference libraries, and etc.

puts "RM-info : Completed script [info script]\n"
```

</details>


<details>
  <summary>icc2_dp_setup.tcl</summary>

</details>




The script sources two setup files containing:
- Common ICC2 variables and paths
- Design planning specific configurations including design name, library paths, and flow control variables

### 2. Design Library Creation

```tcl
if {[file exists ${WORK_DIR}/$DESIGN_LIBRARY]} {
   file delete -force ${WORK_DIR}/${DESIGN_LIBRARY}
}
```

Any existing design library is deleted to ensure a clean run.

The library creation supports multiple technology specification methods:

```tcl
set create_lib_cmd "create_lib ${WORK_DIR}/$DESIGN_LIBRARY"
if {[file exists [which $TECH_FILE]]} {
   lappend create_lib_cmd -tech $TECH_FILE ;# recommended
} elseif {$TECH_LIB != ""} {
   lappend create_lib_cmd -use_technology_lib $TECH_LIB ;# optional
}
lappend create_lib_cmd -ref_libs $REFERENCE_LIBRARY
```

This approach:
- Prefers explicit technology file (`TECH_FILE`) when available
- Falls back to technology library (`TECH_LIB`) if tech file not found
- Always includes reference libraries for standard cells and other design elements

### 3. Netlist Reading

The script supports both hierarchical and flat design flows:

```tcl
if {$DP_FLOW == "hier" && $BOTTOM_BLOCK_VIEW == "abstract"} {
   puts "RM-info : Reading verilog outline (${VERILOG_NETLIST_FILES})"
   read_verilog_outline -design ${DESIGN_NAME}/${INIT_DP_LABEL_NAME} -top ${DESIGN_NAME} ${VERILOG_NETLIST_FILES}
} else {
   puts "RM-info : Reading full chip verilog (${VERILOG_NETLIST_FILES})"
   read_verilog -design ${DESIGN_NAME}/${INIT_DP_LABEL_NAME} -top ${DESIGN_NAME} ${VERILOG_NETLIST_FILES}
}
```

- **Hierarchical flow with abstracts**: Uses `read_verilog_outline` to read design outline for top-level floorplanning
- **Flat or full hierarchical flow**: Uses `read_verilog` to read complete netlist

### 4. Technology Setup

Technology configuration is sourced from an external file when available:

```tcl
if {$TECH_FILE != "" || ($TECH_LIB != "" && !$TECH_LIB_INCLUDES_TECH_SETUP_INFO)} {
   if {[file exists [which $TCL_TECH_SETUP_FILE]]} {
      puts "RM-info : Sourcing [which $TCL_TECH_SETUP_FILE]"
      source -echo $TCL_TECH_SETUP_FILE
   }
}
```

This file typically contains:
- Routing layer direction definitions
- Site definitions and symmetry
- Manufacturing grid and track specifications

### 5. Parasitic Setup

TLU+ parasitic models are configured for accurate RC extraction:

```tcl
if {[file exists [which $TCL_PARASITIC_SETUP_FILE]]} {
   puts "RM-info : Sourcing [which $TCL_PARASITIC_SETUP_FILE]"
   source -echo $TCL_PARASITIC_SETUP_FILE
} else {
   puts "RM-info : No TLU plus files sourced, Parasitic library containing TLU+ must be included in library reference list"
}
```

The parasitic setup file uses `read_parasitic_tech` command to load TLU+ files for different corners.

### 6. Routing Layer Configuration

Maximum and minimum routing layers are constrained:

```tcl
if {$MAX_ROUTING_LAYER != ""} {set_ignored_layers -max_routing_layer $MAX_ROUTING_LAYER}
if {$MIN_ROUTING_LAYER != ""} {set_ignored_layers -min_routing_layer $MIN_ROUTING_LAYER}
```

This ensures routing stays within specified metal layers based on design requirements and technology constraints.

### 7. Pre-Floorplan Design Checks

Design rule checking is performed before floorplanning:

```tcl
if {$CHECK_DESIGN} {
   redirect -file ${REPORTS_DIR_INIT_DP}/check_design.pre_floorplan {
      check_design -ems_database check_design.pre_floorplan.ems -checks dp_pre_floorplan
   }
}
```

This generates a comprehensive report checking:
- Design connectivity and hierarchy
- Reference library consistency
- Technology setup completeness
- Design planning readiness

### 8. Floorplan Initialization

The floorplan is created with explicit die boundary and core offsets:

```tcl
initialize_floorplan \
    -control_type die \
    -boundary {{0 0} {3588 5188}} \
    -core_offset {200 200 200 200}
```

This creates:
- Die area: \(3588 × 5188\) µm²
- Core area: \(3188 × 4788\) µm² (after 200 µm margins)

### 9. Saving the Design

```tcl
save_block -force -label floorplan
save_lib -all
```

The floorplan is saved with label "floorplan" for easy reference, and all libraries are saved to preserve the state.

***

## Key Features

### Modular Setup Approach

The script uses external setup files (`icc2_common_setup.tcl` and `icc2_dp_setup.tcl`) to:
- Centralize variable definitions
- Enable easy customization across different designs
- Maintain consistency in multi-user environments

### Technology Flexibility

Supports multiple technology specification methods:
- Direct technology file (`-tech $TECH_FILE`)
- Technology library reference (`-use_technology_lib $TECH_LIB`)
- Automatic detection and appropriate selection

### Flow Adaptability

Conditional netlist reading supports:
- Hierarchical design flows with abstract views
- Flat design flows
- Top-down design planning approaches

### Quality Checks

Built-in design checking before floorplanning catches:
- Missing or incorrect library references
- Technology setup issues
- Connectivity problems
- Design planning readiness issues

***

## How to Run

### Prerequisites

1. Set up your ICC2 environment with proper licensing
2. Prepare setup files:
   - `icc2_common_setup.tcl` with design name, library paths, and common variables
   - `icc2_dp_setup.tcl` with design planning specific settings
3. Ensure reference libraries and technology files are accessible
4. Create reports directory: `mkdir -p ${REPORTS_DIR_INIT_DP}`

### Execution

From the ICC2 shell:

```tcl
source floorplan_script.tcl
```

Or from the command line:

```bash
icc2_shell -f floorplan_script.tcl | tee floorplan.log
```

### Post-Execution

After successful execution:
- Open the design library in ICC2 GUI for visualization
- Review `check_design.pre_floorplan` report for any warnings
- Verify die and core boundaries match specifications
- Check that the floorplan block is saved with correct label

***

## Verification Checklist

Use this checklist to confirm proper floorplan creation:

- Design library created successfully in `${WORK_DIR}/${DESIGN_LIBRARY}`
- Netlist read without critical errors (warnings about unresolved references may be acceptable)
- Technology setup completed (check ICC2 log for "RM-info" messages)
- Parasitic models loaded successfully
- Pre-floorplan design check passed or has only acceptable warnings
- Die size exactly \(3588 µm × 5188 µm\)
- Core margins of 200 µm on all four sides
- Block saved with label "floorplan"
- All libraries saved successfully

***

## Variable Definitions

Key variables expected in setup files:

### From `icc2_common_setup.tcl` and `icc2_dp_setup.tcl`:

- `DESIGN_NAME`: Top-level design name
- `DESIGN_LIBRARY`: ICC2 design library name
- `WORK_DIR`: Working directory path
- `REFERENCE_LIBRARY`: List of reference NDM libraries
- `TECH_FILE`: Technology file path (optional)
- `TECH_LIB`: Technology library name (optional)
- `VERILOG_NETLIST_FILES`: Path(s) to synthesized Verilog
- `TCL_TECH_SETUP_FILE`: Technology setup TCL script
- `TCL_PARASITIC_SETUP_FILE`: TLU+ setup TCL script
- `MAX_ROUTING_LAYER`: Maximum metal layer for routing
- `MIN_ROUTING_LAYER`: Minimum metal layer for routing
- `DP_FLOW`: Design planning flow type ("hier" or "flat")
- `BOTTOM_BLOCK_VIEW`: View type for hierarchical blocks
- `INIT_DP_LABEL_NAME`: Label name for initial design planning
- `CHECK_DESIGN`: Boolean to enable pre-floorplan checking
- `REPORTS_DIR_INIT_DP`: Directory for reports

***

## Best Practices

1. **Always review setup files** before running to ensure paths and variables are correct
2. **Enable check_design** to catch issues early in the flow
3. **Save intermediate checkpoints** after major steps for debugging
4. **Document any setup file modifications** for reproducibility
5. **Keep consistent naming** across design library, labels, and reports
