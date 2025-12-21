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

### floorplan.tcl
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

```tcl
puts "RM-info : Running script [info script]\n"

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
	
```tcl
puts "RM-info : Running script [info script]\n"
##########################################################################################
# Tool: IC Compiler II 
# Script: icc2_dp_setup.tcl 
# Version: P-2019.03-SP4
# Copyright (C) 2014-2019 Synopsys, Inc. All rights reserved.
##########################################################################################

##########################################################################################
# 				Flow Setup
##########################################################################################
set DP_FLOW         "flat"    ;# hier or flat
set FLOORPLAN_STYLE "channel" ;# Supported design styles channel, abutted
set CHECK_DESIGN    "true"    ;# run atomic check_design pre checks prior to DP commands

set DISTRIBUTED 0  ;# Use distributed runs
### It is required to include the set_host_options command to enable distributed mode tasks. For example,
set_host_options -max_cores 8
#set_host_options -name block_script -submit_command [list qsub -P bnormal -l mem_free=6G,qsc=o -cwd]

set BLOCK_DIST_JOB_FILE             ""     ;# File to set block specific resource requests for distributed jobs
# For example:
#   set_host_options -name block_script  -submit_command "bsub -q normal"
#   set_host_options -name large_block   -submit_command "bsub -q huge"
#   set_host_options -name special_block -submit_command "rsh" local_machine
#   set_app_options -block [get_block block4] -list {plan.distributed_run.block_host large_block}
#   set_app_options -block [get_block block5] -list {plan.distributed_run.block_host large_block}
#   set_app_options -block [get_block block2] -list {plan.distributed_run.block_host special_block}
#  
#   All the jobs associated with blocks that do not have the plan.distributed_run.block_host app option specified
#   will run using the block_script host option. The jobs for blocks block4 and block5 will use the large_block 
#   host option. The job form  block2  will  use  the  special_block host option.


##########################################################################################
# If the design is run with MIBs then change the block list appropriately
##########################################################################################
set DP_BLOCK_REFS                     [list] ;# design names for each physical block (including black boxes) in the design;
                                             ;# this includes bottom and mid level blocks in a Multiple Physical Hierarchy (MPH) design
set DP_INTERMEDIATE_LEVEL_BLOCK_REFS  [list data_memory] ;# design reference names for mid level blocks only
set DP_BB_BLOCK_REFS                  [list] ;# Black Box reference names 
set BOTTOM_BLOCK_VIEW             "abstract" ;# Support abstract or design view for bottom blocks
                                             ;# in the hier flow
set INTERMEDIATE_BLOCK_VIEW       "abstract" ;# Support abstract or design view for intermediate blocks

if { [info exists INTERMEDIATE_BLOCK_VIEW] && $INTERMEDIATE_BLOCK_VIEW == "abstract" } {
   set_app_options -name abstract.allow_all_level_abstract -value true  ;# Dafult value is false
}

# Provide blackbox instanace: target area, BB UPF file, BB Timing file, boundary
#set DP_BB_BLOCK_REFS "leon3s_bb"
#set DP_BB_BLOCKS(leon3s_bb,area)        [list 1346051] ;
#set DP_BB_BLOCKS(leon3s_bb,upf)         [list ${des_dir}/leon3s_bb.upf] ;
#set DP_BB_BLOCKS(leon3s_bb,timing)      [list ${des_dir}/leon3s_bbt.tcl] ;
#set DP_BB_BLOCKS(leon3s_bb,boundary)    { {x1 y1} {x2 y1} {x2 y2} {x1 y2} {x1 y1} } ;
#set DP_BB_SPLIT    "true"


##########################################################################################
# 				CONSTRAINTS / UPF INTENT
##########################################################################################
set TCL_TIMING_RULER_SETUP_FILE       "" ;# file sourced to define parasitic constraints for use with timing ruler 
                                          # before full extraction environment is defined
                                          # Example setup file:
                                          #       set_parasitic_parameters \
                                          #         -early_spec para_WORST \
                                          #         -late_spec para_WORST
set CONSTRAINT_MAPPING_FILE           "" ;# Constraint Mapping File. Default is "split/mapfile"
set TCL_UPF_FILE                      "" ;# Optional power intent TCL script


##########################################################################################
# 				TOP LEVEL FLOORPLAN CREATION (die, pad, RDL) / PLACE IO
##########################################################################################
set TCL_PHYSICAL_CONSTRAINTS_FILE     "" ;# TCL script for primary die area creation. If specified, DEF_FLOORPLAN_FILES will be loaded after TCL_PHYSICAL_CONSTRAINTS_FILE
set TCL_PRE_COMMIT_FILE               "" ;# file sourced to set attributes, lib cell purposes, .. etc on specific cells, prior to running commit_block
set TCL_USER_INIT_DP_POST_SCRIPT      "" ;# An optional Tcl file to be sourced at the very end of init_dp.tcl before save_block.


##########################################################################################
# 				PRE_SHAPING
##########################################################################################
set TCL_USER_PRE_SHAPING_PRE_SCRIPT   "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PRE_SHAPING_POST_SCRIPT  "" ;# An optional Tcl file to be sourced at the very end of the task

##########################################################################################
# 				PLACE_IO
##########################################################################################
set TCL_PAD_CONSTRAINTS_FILE          "/home/prakhan/task_5/icc2_workshop_collaterals/standaloneFlow/pad_placement_constraints.tcl" ;# file sourced to create everything needed by place_io to complete IO placement
                                         ;# including flip chip bumps, and io constraints
set TCL_RDL_FILE                      "" ;# file sourced to create RDL routes
set TCL_USER_PLACE_IO_PRE_SCRIPT      "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PLACE_IO_POST_SCRIPT     "" ;# An optional Tcl file to be sourced at the very end of the task


##########################################################################################
# 				SHAPING
##########################################################################################
switch $FLOORPLAN_STYLE {
   channel {set SHAPING_CMD_OPTIONS           "-channels true"} 
   abutted {set SHAPING_CMD_OPTIONS           "-channels false"} 
}

set TCL_SHAPING_CONSTRAINTS_FILE      "" ;# Specify any constraints prior to shaping i.e. set_shaping_options
                                          # or specify some block shapes manually, for example:
                                          #    set_block_boundary -cell block1 -boundary {{2.10 2.16} {2.10 273.60} \
                                          #    {262.02 273.60} {262.02 2.16}} -orientation R0
                                          #    set_fixed_objects [get_cells block1]
                                          # Support TCL based shaping constraints
                                          # An example is in rm_icc2_dp_scripts/tcl_shaping_constraints_example.tcl
set SHAPING_CONSTRAINTS_FILE          "" ;# Will be included as the -constraint_file option for shape_blocks
set TCL_SHAPING_PNS_STRATEGY_FILE     "" ;# file sourced to create PG strategies for block grid creation
set TCL_MANUAL_SHAPING_FILE           "" ;# File sourced to re-create all block shapes.
                                          # If this file exists, automatic shaping will be by-passed.
                                          # Existing auto or manual block shapes can be written out using the following:
                                          #    write_floorplan -objects <BLOCK_INSTS>
set TCL_USER_SHAPING_PRE_SCRIPT       "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_SHAPING_POST_SCRIPT      "" ;# An optional Tcl file to be sourced at the very end of the task


##########################################################################################
# 				PLACEMENT
##########################################################################################
set TCL_PLACEMENT_CONSTRAINTS_FILE    "" ;# Placeholder for any macro or standard cell placement constraints & options.
                                          # File is sourced prior to DP placement
set PLACEMENT_PIN_CONSTRAINT_AWARE    "false" ;# tells create_placement to consider pin constraints during placement
set USE_INCREMENTAL_DATA              "0" ;# Use floorplan constraints that were written out on a previous run
set CONGESTION_DRIVEN_PLACEMENT       "" ;# Set to one of the following: std_cell, macro, or both to enable congestion driven placement
set TIMING_DRIVEN_PLACEMENT           "" ;# Set to one of the following: std_cell, macro, or both to enable timing driven placement
set TCL_USER_PLACEMENT_PRE_SCRIPT     "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PLACEMENT_POST_SCRIPT    "" ;# An optional Tcl file to be sourced at the very end of the task


##########################################################################################
#				GLOBAL PLANNING
##########################################################################################
set TCL_GLOBAL_PLANNING_FILE          "" ;#Global planning for bus/critical nets


##########################################################################################
# 				PNS
##########################################################################################
set TCL_PNS_FILE                      "./pns_example.tcl" ;# File sourced to define all power structures. 
                                          # This file will include the following types of PG commands:
                                          #   PG Regions
                                          #   PG Patterns
                                          #   PG Strategies
                                          # Note: The file should not contain compile_pg statements
                                          # An example is in rm_icc2_dp_scripts/pns_example.tcl
set PNS_CHARACTERIZE_FLOW             "true"  ;# Perform PG characterization and implementation
set TCL_COMPILE_PG_FILE               "./compile_pg_example.tcl" ;# File should contain all the compile_pg_* commands to create the power networks 
                                          # specified in the strategies in the TCL_PNS_FILE. 
                                          # An example is in rm_icc2_dp_scripts/compile_pg_example.tcl
set TCL_PG_PUSHDOWN_FILE              "" ;# Create this file to facilitate manual pushdown and bypass auto pushdown in the flow.
set TCL_POST_PNS_FILE                 "" ;# If it exists, this file will be sourced after PG creation.
set TCL_USER_CREATE_POWER_PRE_SCRIPT  "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_CREATE_POWER_POST_SCRIPT "" ;# An optional Tcl file to be sourced at the very end of the task


##########################################################################################
# 				PLACE PINS
##########################################################################################
##Note:Feedthroughs are disabled by default. Enable feedthroughs either through set_*_pin_constraints  Tcl commands or through Pin constraints file
set TCL_PIN_CONSTRAINT_FILE           "" ;# file sourced to apply set_*_pin_constraints to the design
set CUSTOM_PIN_CONSTRAINT_FILE        "" ;# will be loaded via read_pin_constraints -file
                                         ;# used for more complex pin constraints, 
                                         ;# or in constraint replay
set PLACE_PINS_SELF                   "true" ;# Set to true if the block's top level pins are not all connected to IO drivers.
set TIMING_PIN_PLACEMENT              "true" ;# Set to true for timing driven pin placement
set TCL_USER_PLACE_PINS_PRE_SCRIPT    "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PLACE_PINS_POST_SCRIPT   "" ;# An optional Tcl file to be sourced at the very end of the task


##########################################################################################
# 				PRE-TIMING
##########################################################################################
set TCL_USER_PRE_TIMING_PRE_SCRIPT    "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PRE_TIMING_POST_SCRIPT   "" ;# An optional Tcl file to be sourced at the very end of the task


##########################################################################################
# 				TIMING ESTIMATION
##########################################################################################
set TCL_TIMING_ESTIMATION_SETUP_FILE       "" ;# Specify any constraints prior to timing estimation
set TCL_USER_TIMING_ESTIMATION_PRE_SCRIPT  "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_TIMING_ESTIMATION_POST_SCRIPT "" ;# An optional Tcl file to be sourced at the very end of the task
set TCL_LIB_CELL_DONT_USE_FILE             "" ;# A Tcl file for customized don't use ("set_lib_cell_purpose -exclude <purpose>" commands);
                                              ;#  to prevent estimate_timing picking a lib_cell not used by pnr flow.
                                              ;#  please specify non-optimization purpose lib cells in the file. 


##########################################################################################
# 				BUDGETING
##########################################################################################
set TCL_BUDGETING_SETUP_FILE                "" ;# Specify any constraints prior to budgeting
set TCL_BOUNDARY_BUDGETING_CONSTRAINTS_FILE "" ;# An optional user constraints file to override compute_budget_constraints
set TCL_USER_BUDGETING_PRE_SCRIPT           "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_BUDGETING_POST_SCRIPT          "" ;# An optional Tcl file to be sourced at the very end of the task


##########################################################################################
## System Variables (there's no need to change the following)
##########################################################################################
set WORK_DIR ./work
set WORK_DIR_WRITE_DATA ./write_data_dir
if !{[file exists $WORK_DIR]} {file mkdir $WORK_DIR}

set SPLIT_CONSTRAINTS_LABEL_NAME split_constraints
set INIT_DP_LABEL_NAME init_dp
set PRE_SHAPING_LABEL_NAME pre_shaping
set PLACE_IO_LABEL_NAME place_io
set SHAPING_LABEL_NAME shaping
set PLACEMENT_LABEL_NAME placement
set CREATE_POWER_LABEL_NAME create_power
set CLOCK_TRUNK_PLANNING_LABEL_NAME clock_trunk_planning
set PLACE_PINS_LABEL_NAME place_pins
set PRE_TIMING_LABEL_NAME pre_timing
set TIMING_ESTIMATION_LABEL_NAME timing_estimation
set BUDGETING_LABEL_NAME budgeting

# Block label to be release by write_data
if {$DP_FLOW == "flat"} {
   set WRITE_DATA_LABEL_NAME timing_estimation
} else {
   set WRITE_DATA_LABEL_NAME budgeting
}

## Directories
set OUTPUTS_DIR	"./outputs_icc2"	;# Directory to write output data files; mainly used by write_data.tcl
set REPORTS_DIR	"./rpts_icc2"		;# Directory to write reports; mainly used by report_qor.tcl

set REPORTS_DIR_SPLIT_CONSTRAINTS $REPORTS_DIR/$SPLIT_CONSTRAINTS_LABEL_NAME
set REPORTS_DIR_INIT_DP $REPORTS_DIR/$INIT_DP_LABEL_NAME
set REPORTS_DIR_PRE_SHAPING $REPORTS_DIR/$PRE_SHAPING_LABEL_NAME
set REPORTS_DIR_PLACE_IO $REPORTS_DIR/$PLACE_IO_LABEL_NAME
set REPORTS_DIR_SHAPING $REPORTS_DIR/$SHAPING_LABEL_NAME
set REPORTS_DIR_PLACEMENT $REPORTS_DIR/$PLACEMENT_LABEL_NAME
set REPORTS_DIR_CREATE_POWER $REPORTS_DIR/$CREATE_POWER_LABEL_NAME
set REPORTS_DIR_CLOCK_TRUNK_PLANNING $REPORTS_DIR/$CLOCK_TRUNK_PLANNING_LABEL_NAME
set REPORTS_DIR_PLACE_PINS $REPORTS_DIR/$PLACE_PINS_LABEL_NAME
set REPORTS_DIR_PRE_TIMING $REPORTS_DIR/$PRE_TIMING_LABEL_NAME
set REPORTS_DIR_TIMING_ESTIMATION $REPORTS_DIR/$TIMING_ESTIMATION_LABEL_NAME
set REPORTS_DIR_BUDGETING $REPORTS_DIR/$BUDGETING_LABEL_NAME

if !{[file exists $REPORTS_DIR]} {file mkdir $REPORTS_DIR}
if !{[file exists $OUTPUTS_DIR]} {file mkdir $OUTPUTS_DIR}
if !{[file exists $REPORTS_DIR_SPLIT_CONSTRAINTS]} {file mkdir $REPORTS_DIR_SPLIT_CONSTRAINTS}
if !{[file exists $REPORTS_DIR_INIT_DP]} {file mkdir $REPORTS_DIR_INIT_DP}
if !{[file exists $REPORTS_DIR_PRE_SHAPING]} {file mkdir $REPORTS_DIR_PRE_SHAPING}
if !{[file exists $REPORTS_DIR_PLACE_IO]} {file mkdir $REPORTS_DIR_PLACE_IO}
if !{[file exists $REPORTS_DIR_SHAPING]} {file mkdir $REPORTS_DIR_SHAPING}
if !{[file exists $REPORTS_DIR_PLACEMENT]} {file mkdir $REPORTS_DIR_PLACEMENT}
if !{[file exists $REPORTS_DIR_CREATE_POWER]} {file mkdir $REPORTS_DIR_CREATE_POWER}
if !{[file exists $REPORTS_DIR_CLOCK_TRUNK_PLANNING]} {file mkdir $REPORTS_DIR_CLOCK_TRUNK_PLANNING}
if !{[file exists $REPORTS_DIR_PLACE_PINS]} {file mkdir $REPORTS_DIR_PLACE_PINS}
if !{[file exists $REPORTS_DIR_PRE_TIMING]} {file mkdir $REPORTS_DIR_PRE_TIMING}
if !{[file exists $REPORTS_DIR_TIMING_ESTIMATION]} {file mkdir $REPORTS_DIR_TIMING_ESTIMATION}
if !{[file exists $REPORTS_DIR_BUDGETING]} {file mkdir $REPORTS_DIR_BUDGETING}


if {[info exists env(LOGS_DIR)]} {
   set log_dir $env(LOGS_DIR)
} else {
   set log_dir ./logs_icc2 
}


set search_path [list ./rm_icc2_dp_scripts ./rm_icc2_pnr_scripts $WORK_DIR ] 
lappend search_path .

##########################################################################################
# 				Optional Settings
##########################################################################################
set_message_info -id PVT-012 -limit 1
set_message_info -id PVT-013 -limit 1
set search_path "/home/prakhan/task_5/icc2_workshop_collaterals/"
set_app_var link_library "nangate_typical.db sram_32_1024_freepdk45_TT_1p0V_25C_lib.db"

puts "RM-info : Completed script [info script]\n"
```
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

In terminal :

```bash
csh
source ~/toolRC_iitgntapeout
icc2_shell -f floorplan.tcl
```

### Post-Execution

<img width="1602" height="1019" alt="image" src="https://github.com/user-attachments/assets/52b5adb6-7078-4e05-a364-f8d5318eeee2" />

After successful execution:
- Open the design library in ICC2 GUI for visualization

```
gui_show_man_page
```

<img width="1679" height="1046" alt="image" src="https://github.com/user-attachments/assets/18c34f60-0c37-4dc9-8893-b21156a5d2d7" />

```
get_ports
```

<img width="1678" height="1043" alt="image" src="https://github.com/user-attachments/assets/03dfd4ed-11ba-4ec2-a7b8-e6e56b133a3a" />

```
place_pins -self
```
<img width="1679" height="1046" alt="image" src="https://github.com/user-attachments/assets/79b9eb41-9c35-4d4d-8c02-809dbb03a19d" />

***

```
write_def fp_raven.def
```

<img width="675" height="617" alt="image" src="https://github.com/user-attachments/assets/d379efb3-8e38-49df-80cc-6a9532d1aece" />


## Report & Log

<details>
  <summary>Report</summary>

```
===== FLOORPLAN GEOMETRY (USER DEFINED) =====
Die Area  : 0 0 3588 5188  (microns)
Core Area : 200 200 3388 4988  (microns)

===== TOP LEVEL PORTS =====
{vddio vddio_2 vssio vssio_2 vdda vssa vccd vssd vdda1 vdda1_2 vdda2 vssa1 vssa1_2 vssa2 vccd1 vccd2 vssd1 vssd2 gpio {mprj_io[37]} {mprj_io[36]} {mprj_io[35]} {mprj_io[34]} {mprj_io[33]} {mprj_io[32]} {mprj_io[31]} {mprj_io[30]} {mprj_io[29]} {mprj_io[28]} {mprj_io[27]} {mprj_io[26]} {mprj_io[25]} {mprj_io[24]} {mprj_io[23]} {mprj_io[22]} {mprj_io[21]} {mprj_io[20]} {mprj_io[19]} {mprj_io[18]} {mprj_io[17]} {mprj_io[16]} {mprj_io[15]} {mprj_io[14]} {mprj_io[13]} {mprj_io[12]} {mprj_io[11]} {mprj_io[10]} {mprj_io[9]} {mprj_io[8]} {mprj_io[7]} {mprj_io[6]} {mprj_io[5]} {mprj_io[4]} {mprj_io[3]} {mprj_io[2]} {mprj_io[1]} {mprj_io[0]} clock resetb flash_csb flash_clk flash_io0 flash_io1}
```
</details>

<details>
  <summary>Log</summary>
	
```
[SCL] 12/21/2025 12:57:48 PID:12598 Client:nanodc.iitgn.ac.in Authorization failed Synopsys 2022.12
[SCL] 12/21/2025 12:57:49 PID:12598 Client:nanodc.iitgn.ac.in Authorization failed Synopsys-Release 2022.12
[SCL] 12/21/2025 12:57:49 PID:12598 Client:nanodc.iitgn.ac.in Authorization failed Galaxy-Internal-Debug 2022.12
[icc2-lic Sun Dec 21 12:57:49 2025] Sending authorization request for 'ICCompilerII'
[SCL] 12/21/2025 12:57:50 PID:12598 Client:nanodc.iitgn.ac.in Authorization failed ICCompilerII 2022.12
[icc2-lic Sun Dec 21 12:57:50 2025] Authorization request for 'ICCompilerII' failed (-5)
[icc2-lic Sun Dec 21 12:57:50 2025] Sending authorization request for 'ICCompilerII-NX'
[SCL] 12/21/2025 12:57:51 PID:12598 Client:nanodc.iitgn.ac.in Server:27020@c2s.cdacb.in Authorization succeeded ICCompilerII-NX 2022.12
[icc2-lic Sun Dec 21 12:57:51 2025] Authorization request for 'ICCompilerII-NX' succeeded



                              IC Compiler II (TM)

                Version U-2022.12-SP3 for linux64 - Apr 18, 2023
  This release has significant feature enhancements. Please review the Release
                       Notes associated with this release.

                    Copyright (c) 1988 - 2023 Synopsys, Inc.
   This software and the associated documentation are proprietary to Synopsys,
 Inc. This software may only be used in accordance with the terms and conditions
 of a written license agreement with Synopsys, Inc. All other use, reproduction,
   or distribution of this software is strictly prohibited.  Licensed Products
     communicate with Synopsys servers for the purpose of providing software
    updates, detecting software piracy and verifying that customers are using
    Licensed Products in conformity with the applicable License Key for such
  Licensed Products. Synopsys will use information gathered in connection with
    this process to deliver software updates and pursue software pirates and
                                   infringers.

 Inclusivity & Diversity - Visit SolvNetPlus to read the "Synopsys Statement on
            Inclusivity and Diversity" (Refer to article 000036315 at
                        https://solvnetplus.synopsys.com)
 
source -echo /home/prakhan/task_5/icc2_workshop_collaterals/standaloneFlow/icc2_common_setup.tcl
puts "RM-info : Running script [info script]\n"
RM-info : Running script /home/prakhan/task_5/icc2_workshop_collaterals/standaloneFlow/icc2_common_setup.tcl

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
RM-info : Completed script /home/prakhan/task_5/icc2_workshop_collaterals/standaloneFlow/icc2_common_setup.tcl

source -echo /home/prakhan/task_5/icc2_workshop_collaterals/standaloneFlow/icc2_dp_setup.tcl
puts "RM-info : Running script [info script]\n"
RM-info : Running script /home/prakhan/task_5/icc2_workshop_collaterals/standaloneFlow/icc2_dp_setup.tcl

##########################################################################################
# Tool: IC Compiler II 
# Script: icc2_dp_setup.tcl 
# Version: P-2019.03-SP4
# Copyright (C) 2014-2019 Synopsys, Inc. All rights reserved.
##########################################################################################
##########################################################################################
# 				Flow Setup
##########################################################################################
set DP_FLOW         "flat"    ;# hier or flat
set FLOORPLAN_STYLE "channel" ;# Supported design styles channel, abutted
set CHECK_DESIGN    "true"    ;# run atomic check_design pre checks prior to DP commands
set DISTRIBUTED 0  ;# Use distributed runs
### It is required to include the set_host_options command to enable distributed mode tasks. For example,
set_host_options -max_cores 8
#set_host_options -name block_script -submit_command [list qsub -P bnormal -l mem_free=6G,qsc=o -cwd]
set BLOCK_DIST_JOB_FILE             ""     ;# File to set block specific resource requests for distributed jobs
# For example:
#   set_host_options -name block_script  -submit_command "bsub -q normal"
#   set_host_options -name large_block   -submit_command "bsub -q huge"
#   set_host_options -name special_block -submit_command "rsh" local_machine
#   set_app_options -block [get_block block4] -list {plan.distributed_run.block_host large_block}
#   set_app_options -block [get_block block5] -list {plan.distributed_run.block_host large_block}
#   set_app_options -block [get_block block2] -list {plan.distributed_run.block_host special_block}
#  
#   All the jobs associated with blocks that do not have the plan.distributed_run.block_host app option specified
#   will run using the block_script host option. The jobs for blocks block4 and block5 will use the large_block 
#   host option. The job form  block2  will  use  the  special_block host option.
##########################################################################################
# If the design is run with MIBs then change the block list appropriately
##########################################################################################
set DP_BLOCK_REFS                     [list] ;# design names for each physical block (including black boxes) in the design;
;# this includes bottom and mid level blocks in a Multiple Physical Hierarchy (MPH) design
set DP_INTERMEDIATE_LEVEL_BLOCK_REFS  [list data_memory] ;# design reference names for mid level blocks only
set DP_BB_BLOCK_REFS                  [list] ;# Black Box reference names 
set BOTTOM_BLOCK_VIEW             "abstract" ;# Support abstract or design view for bottom blocks
;# in the hier flow
set INTERMEDIATE_BLOCK_VIEW       "abstract" ;# Support abstract or design view for intermediate blocks
if { [info exists INTERMEDIATE_BLOCK_VIEW] && $INTERMEDIATE_BLOCK_VIEW == "abstract" } {
   set_app_options -name abstract.allow_all_level_abstract -value true  ;# Dafult value is false
}
Warning: Application option 'abstract.allow_all_level_abstract' will be made block-scoped in the upcoming 2019.12 release. (ABS-549)
# Provide blackbox instanace: target area, BB UPF file, BB Timing file, boundary
#set DP_BB_BLOCK_REFS "leon3s_bb"
#set DP_BB_BLOCKS(leon3s_bb,area)        [list 1346051] ;
#set DP_BB_BLOCKS(leon3s_bb,upf)         [list ${des_dir}/leon3s_bb.upf] ;
#set DP_BB_BLOCKS(leon3s_bb,timing)      [list ${des_dir}/leon3s_bbt.tcl] ;
#set DP_BB_BLOCKS(leon3s_bb,boundary)    { {x1 y1} {x2 y1} {x2 y2} {x1 y2} {x1 y1} } ;
#set DP_BB_SPLIT    "true"
##########################################################################################
# 				CONSTRAINTS / UPF INTENT
##########################################################################################
set TCL_TIMING_RULER_SETUP_FILE       "" ;# file sourced to define parasitic constraints for use with timing ruler 
# before full extraction environment is defined
# Example setup file:
#       set_parasitic_parameters \
                                          #         -early_spec para_WORST \
                                          #         -late_spec para_WORST
set CONSTRAINT_MAPPING_FILE           "" ;# Constraint Mapping File. Default is "split/mapfile"
set TCL_UPF_FILE                      "" ;# Optional power intent TCL script
##########################################################################################
# 				TOP LEVEL FLOORPLAN CREATION (die, pad, RDL) / PLACE IO
##########################################################################################
set TCL_PHYSICAL_CONSTRAINTS_FILE     "" ;# TCL script for primary die area creation. If specified, DEF_FLOORPLAN_FILES will be loaded after TCL_PHYSICAL_CONSTRAINTS_FILE
set TCL_PRE_COMMIT_FILE               "" ;# file sourced to set attributes, lib cell purposes, .. etc on specific cells, prior to running commit_block
set TCL_USER_INIT_DP_POST_SCRIPT      "" ;# An optional Tcl file to be sourced at the very end of init_dp.tcl before save_block.
##########################################################################################
# 				PRE_SHAPING
##########################################################################################
set TCL_USER_PRE_SHAPING_PRE_SCRIPT   "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PRE_SHAPING_POST_SCRIPT  "" ;# An optional Tcl file to be sourced at the very end of the task
##########################################################################################
# 				PLACE_IO
##########################################################################################
set TCL_PAD_CONSTRAINTS_FILE          "/home/prakhan/task_5/icc2_workshop_collaterals/standaloneFlow/pad_placement_constraints.tcl" ;# file sourced to create everything needed by place_io to complete IO placement
;# including flip chip bumps, and io constraints
set TCL_RDL_FILE                      "" ;# file sourced to create RDL routes
set TCL_USER_PLACE_IO_PRE_SCRIPT      "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PLACE_IO_POST_SCRIPT     "" ;# An optional Tcl file to be sourced at the very end of the task
##########################################################################################
# 				SHAPING
##########################################################################################
switch $FLOORPLAN_STYLE {
   channel {set SHAPING_CMD_OPTIONS           "-channels true"} 
   abutted {set SHAPING_CMD_OPTIONS           "-channels false"} 
}
set TCL_SHAPING_CONSTRAINTS_FILE      "" ;# Specify any constraints prior to shaping i.e. set_shaping_options
# or specify some block shapes manually, for example:
#    set_block_boundary -cell block1 -boundary {{2.10 2.16} {2.10 273.60} \
                                          #    {262.02 273.60} {262.02 2.16}} -orientation R0
#    set_fixed_objects [get_cells block1]
# Support TCL based shaping constraints
# An example is in rm_icc2_dp_scripts/tcl_shaping_constraints_example.tcl
set SHAPING_CONSTRAINTS_FILE          "" ;# Will be included as the -constraint_file option for shape_blocks
set TCL_SHAPING_PNS_STRATEGY_FILE     "" ;# file sourced to create PG strategies for block grid creation
set TCL_MANUAL_SHAPING_FILE           "" ;# File sourced to re-create all block shapes.
# If this file exists, automatic shaping will be by-passed.
# Existing auto or manual block shapes can be written out using the following:
#    write_floorplan -objects <BLOCK_INSTS>
set TCL_USER_SHAPING_PRE_SCRIPT       "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_SHAPING_POST_SCRIPT      "" ;# An optional Tcl file to be sourced at the very end of the task
##########################################################################################
# 				PLACEMENT
##########################################################################################
set TCL_PLACEMENT_CONSTRAINTS_FILE    "" ;# Placeholder for any macro or standard cell placement constraints & options.
# File is sourced prior to DP placement
set PLACEMENT_PIN_CONSTRAINT_AWARE    "false" ;# tells create_placement to consider pin constraints during placement
set USE_INCREMENTAL_DATA              "0" ;# Use floorplan constraints that were written out on a previous run
set CONGESTION_DRIVEN_PLACEMENT       "" ;# Set to one of the following: std_cell, macro, or both to enable congestion driven placement
set TIMING_DRIVEN_PLACEMENT           "" ;# Set to one of the following: std_cell, macro, or both to enable timing driven placement
set TCL_USER_PLACEMENT_PRE_SCRIPT     "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PLACEMENT_POST_SCRIPT    "" ;# An optional Tcl file to be sourced at the very end of the task
##########################################################################################
#				GLOBAL PLANNING
##########################################################################################
set TCL_GLOBAL_PLANNING_FILE          "" ;#Global planning for bus/critical nets
##########################################################################################
# 				PNS
##########################################################################################
set TCL_PNS_FILE                      "./pns_example.tcl" ;# File sourced to define all power structures. 
# This file will include the following types of PG commands:
#   PG Regions
#   PG Patterns
#   PG Strategies
# Note: The file should not contain compile_pg statements
# An example is in rm_icc2_dp_scripts/pns_example.tcl
set PNS_CHARACTERIZE_FLOW             "true"  ;# Perform PG characterization and implementation
set TCL_COMPILE_PG_FILE               "./compile_pg_example.tcl" ;# File should contain all the compile_pg_* commands to create the power networks 
# specified in the strategies in the TCL_PNS_FILE. 
# An example is in rm_icc2_dp_scripts/compile_pg_example.tcl
set TCL_PG_PUSHDOWN_FILE              "" ;# Create this file to facilitate manual pushdown and bypass auto pushdown in the flow.
set TCL_POST_PNS_FILE                 "" ;# If it exists, this file will be sourced after PG creation.
set TCL_USER_CREATE_POWER_PRE_SCRIPT  "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_CREATE_POWER_POST_SCRIPT "" ;# An optional Tcl file to be sourced at the very end of the task
##########################################################################################
# 				PLACE PINS
##########################################################################################
##Note:Feedthroughs are disabled by default. Enable feedthroughs either through set_*_pin_constraints  Tcl commands or through Pin constraints file
set TCL_PIN_CONSTRAINT_FILE           "" ;# file sourced to apply set_*_pin_constraints to the design
set CUSTOM_PIN_CONSTRAINT_FILE        "" ;# will be loaded via read_pin_constraints -file
;# used for more complex pin constraints, 
;# or in constraint replay
set PLACE_PINS_SELF                   "true" ;# Set to true if the block's top level pins are not all connected to IO drivers.
set TIMING_PIN_PLACEMENT              "true" ;# Set to true for timing driven pin placement
set TCL_USER_PLACE_PINS_PRE_SCRIPT    "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PLACE_PINS_POST_SCRIPT   "" ;# An optional Tcl file to be sourced at the very end of the task
##########################################################################################
# 				PRE-TIMING
##########################################################################################
set TCL_USER_PRE_TIMING_PRE_SCRIPT    "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_PRE_TIMING_POST_SCRIPT   "" ;# An optional Tcl file to be sourced at the very end of the task
##########################################################################################
# 				TIMING ESTIMATION
##########################################################################################
set TCL_TIMING_ESTIMATION_SETUP_FILE       "" ;# Specify any constraints prior to timing estimation
set TCL_USER_TIMING_ESTIMATION_PRE_SCRIPT  "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_TIMING_ESTIMATION_POST_SCRIPT "" ;# An optional Tcl file to be sourced at the very end of the task
set TCL_LIB_CELL_DONT_USE_FILE             "" ;# A Tcl file for customized don't use ("set_lib_cell_purpose -exclude <purpose>" commands);
;#  to prevent estimate_timing picking a lib_cell not used by pnr flow.
;#  please specify non-optimization purpose lib cells in the file. 
##########################################################################################
# 				BUDGETING
##########################################################################################
set TCL_BUDGETING_SETUP_FILE                "" ;# Specify any constraints prior to budgeting
set TCL_BOUNDARY_BUDGETING_CONSTRAINTS_FILE "" ;# An optional user constraints file to override compute_budget_constraints
set TCL_USER_BUDGETING_PRE_SCRIPT           "" ;# An optional Tcl file to be sourced at the very beginning of the task
set TCL_USER_BUDGETING_POST_SCRIPT          "" ;# An optional Tcl file to be sourced at the very end of the task
##########################################################################################
## System Variables (there's no need to change the following)
##########################################################################################
set WORK_DIR ./work
set WORK_DIR_WRITE_DATA ./write_data_dir
if !{[file exists $WORK_DIR]} {file mkdir $WORK_DIR}
set SPLIT_CONSTRAINTS_LABEL_NAME split_constraints
set INIT_DP_LABEL_NAME init_dp
set PRE_SHAPING_LABEL_NAME pre_shaping
set PLACE_IO_LABEL_NAME place_io
set SHAPING_LABEL_NAME shaping
set PLACEMENT_LABEL_NAME placement
set CREATE_POWER_LABEL_NAME create_power
set CLOCK_TRUNK_PLANNING_LABEL_NAME clock_trunk_planning
set PLACE_PINS_LABEL_NAME place_pins
set PRE_TIMING_LABEL_NAME pre_timing
set TIMING_ESTIMATION_LABEL_NAME timing_estimation
set BUDGETING_LABEL_NAME budgeting
# Block label to be release by write_data
if {$DP_FLOW == "flat"} {
   set WRITE_DATA_LABEL_NAME timing_estimation
} else {
   set WRITE_DATA_LABEL_NAME budgeting
}
## Directories
set OUTPUTS_DIR	"./outputs_icc2"	;# Directory to write output data files; mainly used by write_data.tcl
set REPORTS_DIR	"./rpts_icc2"		;# Directory to write reports; mainly used by report_qor.tcl
set REPORTS_DIR_SPLIT_CONSTRAINTS $REPORTS_DIR/$SPLIT_CONSTRAINTS_LABEL_NAME
set REPORTS_DIR_INIT_DP $REPORTS_DIR/$INIT_DP_LABEL_NAME
set REPORTS_DIR_PRE_SHAPING $REPORTS_DIR/$PRE_SHAPING_LABEL_NAME
set REPORTS_DIR_PLACE_IO $REPORTS_DIR/$PLACE_IO_LABEL_NAME
set REPORTS_DIR_SHAPING $REPORTS_DIR/$SHAPING_LABEL_NAME
set REPORTS_DIR_PLACEMENT $REPORTS_DIR/$PLACEMENT_LABEL_NAME
set REPORTS_DIR_CREATE_POWER $REPORTS_DIR/$CREATE_POWER_LABEL_NAME
set REPORTS_DIR_CLOCK_TRUNK_PLANNING $REPORTS_DIR/$CLOCK_TRUNK_PLANNING_LABEL_NAME
set REPORTS_DIR_PLACE_PINS $REPORTS_DIR/$PLACE_PINS_LABEL_NAME
set REPORTS_DIR_PRE_TIMING $REPORTS_DIR/$PRE_TIMING_LABEL_NAME
set REPORTS_DIR_TIMING_ESTIMATION $REPORTS_DIR/$TIMING_ESTIMATION_LABEL_NAME
set REPORTS_DIR_BUDGETING $REPORTS_DIR/$BUDGETING_LABEL_NAME
if !{[file exists $REPORTS_DIR]} {file mkdir $REPORTS_DIR}
if !{[file exists $OUTPUTS_DIR]} {file mkdir $OUTPUTS_DIR}
if !{[file exists $REPORTS_DIR_SPLIT_CONSTRAINTS]} {file mkdir $REPORTS_DIR_SPLIT_CONSTRAINTS}
if !{[file exists $REPORTS_DIR_INIT_DP]} {file mkdir $REPORTS_DIR_INIT_DP}
if !{[file exists $REPORTS_DIR_PRE_SHAPING]} {file mkdir $REPORTS_DIR_PRE_SHAPING}
if !{[file exists $REPORTS_DIR_PLACE_IO]} {file mkdir $REPORTS_DIR_PLACE_IO}
if !{[file exists $REPORTS_DIR_SHAPING]} {file mkdir $REPORTS_DIR_SHAPING}
if !{[file exists $REPORTS_DIR_PLACEMENT]} {file mkdir $REPORTS_DIR_PLACEMENT}
if !{[file exists $REPORTS_DIR_CREATE_POWER]} {file mkdir $REPORTS_DIR_CREATE_POWER}
if !{[file exists $REPORTS_DIR_CLOCK_TRUNK_PLANNING]} {file mkdir $REPORTS_DIR_CLOCK_TRUNK_PLANNING}
if !{[file exists $REPORTS_DIR_PLACE_PINS]} {file mkdir $REPORTS_DIR_PLACE_PINS}
if !{[file exists $REPORTS_DIR_PRE_TIMING]} {file mkdir $REPORTS_DIR_PRE_TIMING}
if !{[file exists $REPORTS_DIR_TIMING_ESTIMATION]} {file mkdir $REPORTS_DIR_TIMING_ESTIMATION}
if !{[file exists $REPORTS_DIR_BUDGETING]} {file mkdir $REPORTS_DIR_BUDGETING}
if {[info exists env(LOGS_DIR)]} {
   set log_dir $env(LOGS_DIR)
} else {
   set log_dir ./logs_icc2 
}
set search_path [list ./rm_icc2_dp_scripts ./rm_icc2_pnr_scripts $WORK_DIR ] 
lappend search_path .
##########################################################################################
# 				Optional Settings
##########################################################################################
set_message_info -id PVT-012 -limit 1
set_message_info -id PVT-013 -limit 1
set search_path "/home/prakhan/task_5/icc2_workshop_collaterals/"
set_app_var link_library "nangate_typical.db sram_32_1024_freepdk45_TT_1p0V_25C_lib.db"
puts "RM-info : Completed script [info script]\n"
RM-info : Completed script /home/prakhan/task_5/icc2_workshop_collaterals/standaloneFlow/icc2_dp_setup.tcl

if {[file exists ${WORK_DIR}/$DESIGN_LIBRARY]} {
   file delete -force ${WORK_DIR}/${DESIGN_LIBRARY}
}
###---NDM Library creation---###
set create_lib_cmd "create_lib ${WORK_DIR}/$DESIGN_LIBRARY"
create_lib ./work/raven_wrapperNangate
if {[file exists [which $TECH_FILE]]} {
   lappend create_lib_cmd -tech $TECH_FILE ;# recommended
} elseif {$TECH_LIB != ""} {
   lappend create_lib_cmd -use_technology_lib $TECH_LIB ;# optional
}
create_lib ./work/raven_wrapperNangate -tech /home/prakhan/task_5/icc2_workshop_collaterals/nangate.tf
lappend create_lib_cmd -ref_libs $REFERENCE_LIBRARY
create_lib ./work/raven_wrapperNangate -tech /home/prakhan/task_5/icc2_workshop_collaterals/nangate.tf -ref_libs {/home/prakhan/task_5/icc2_workshop_collaterals/nangate_stdcell.lef /home/prakhan/task_5/icc2_workshop_collaterals/sram/sram_32_1024_freepdk45.lef}
puts "RM-info : $create_lib_cmd"
RM-info : create_lib ./work/raven_wrapperNangate -tech /home/prakhan/task_5/icc2_workshop_collaterals/nangate.tf -ref_libs {/home/prakhan/task_5/icc2_workshop_collaterals/nangate_stdcell.lef /home/prakhan/task_5/icc2_workshop_collaterals/sram/sram_32_1024_freepdk45.lef}
eval ${create_lib_cmd}
Warning: Layer 'metal1' is missing the attribute 'minArea'. (nangate.tf line 79) (TECH-026)
Warning: Layer 'metal2' is missing the attribute 'minArea'. (nangate.tf line 128) (TECH-026)
Warning: Layer 'metal3' is missing the attribute 'minArea'. (nangate.tf line 187) (TECH-026)
Warning: Layer 'metal4' is missing the attribute 'minArea'. (nangate.tf line 246) (TECH-026)
Warning: Layer 'metal5' is missing the attribute 'minArea'. (nangate.tf line 304) (TECH-026)
Warning: Layer 'metal6' is missing the attribute 'minArea'. (nangate.tf line 362) (TECH-026)
Warning: Layer 'metal7' is missing the attribute 'minArea'. (nangate.tf line 420) (TECH-026)
Warning: Layer 'metal8' is missing the attribute 'minArea'. (nangate.tf line 477) (TECH-026)
Warning: Layer 'metal9' is missing the attribute 'minArea'. (nangate.tf line 534) (TECH-026)
Warning: Layer 'metal10' is missing the attribute 'minArea'. (nangate.tf line 590) (TECH-026)
Warning: nangate.tf line 850, ContactCode 'via4_0' has undefined or zero-enclosures. (TECH-250)
Warning: nangate.tf line 868, ContactCode 'via5_0' has undefined or zero-enclosures. (TECH-250)
Warning: nangate.tf line 904, ContactCode 'via7_0' has undefined or zero-enclosures. (TECH-250)
Warning: nangate.tf line 940, ContactCode 'via9_0' has undefined or zero-enclosures. (TECH-250)
Warning: nangate.tf line 1358, ContactCode 'Via4Array-0' has undefined or zero-enclosures. (TECH-250)
Warning: nangate.tf line 1377, ContactCode 'Via5Array-0' has undefined or zero-enclosures. (TECH-250)
Warning: nangate.tf line 1415, ContactCode 'Via7Array-0' has undefined or zero-enclosures. (TECH-250)
Warning: nangate.tf line 1453, ContactCode 'Via9Array-0' has undefined or zero-enclosures. (TECH-250)
Warning: Layer 'via1/via2' is missing the attribute 'maxStackLevel'. (nangate.tf line 1463) (TECH-026)
Warning: Layer 'via2/via3' is missing the attribute 'maxStackLevel'. (nangate.tf line 1472) (TECH-026)
Warning: Layer 'via3/via4' is missing the attribute 'maxStackLevel'. (nangate.tf line 1481) (TECH-026)
Warning: Layer 'via4/via5' is missing the attribute 'maxStackLevel'. (nangate.tf line 1490) (TECH-026)
Warning: Layer 'via5/via6' is missing the attribute 'maxStackLevel'. (nangate.tf line 1499) (TECH-026)
Warning: Layer 'via6/via7' is missing the attribute 'maxStackLevel'. (nangate.tf line 1508) (TECH-026)
Warning: Layer 'via7/via8' is missing the attribute 'maxStackLevel'. (nangate.tf line 1517) (TECH-026)
Warning: Layer 'via8/via9' is missing the attribute 'maxStackLevel'. (nangate.tf line 1526) (TECH-026)
Warning: Layer 'metal1' is missing the attribute 'minArea'. ( line 105) (TECH-026)
Warning: Layer 'metal2' is missing the attribute 'minArea'. ( line 164) (TECH-026)
Warning: Layer 'metal3' is missing the attribute 'minArea'. ( line 223) (TECH-026)
Warning: Layer 'metal4' is missing the attribute 'minArea'. ( line 281) (TECH-026)
Warning: Layer 'metal5' is missing the attribute 'minArea'. ( line 339) (TECH-026)
Warning: Layer 'metal6' is missing the attribute 'minArea'. ( line 397) (TECH-026)
Warning: Layer 'metal7' is missing the attribute 'minArea'. ( line 454) (TECH-026)
Warning: Layer 'metal8' is missing the attribute 'minArea'. ( line 511) (TECH-026)
Warning: Layer 'metal9' is missing the attribute 'minArea'. ( line 567) (TECH-026)
Warning: Layer 'metal10' is missing the attribute 'minArea'. ( line 623) (TECH-026)
Warning: Cut layer 'via1' has a non-cross primary default ContactCode 'via1_4'. (line 628) (TECH-083w)
Warning: Cut layer 'via2' has a non-cross primary default ContactCode 'via2_8'. (line 718) (TECH-083w)
Information: Loading technology file '/home/prakhan/task_5/icc2_workshop_collaterals/nangate.tf' (FILE-007)
...Created 2 lib groups

... 2 cell libraries to build.


... checking whether need to build cell library: NangateOpenCellLibrary.ndm (1/2)
... checking whether can reuse library under output directory
cannot reuse CLIBs/NangateOpenCellLibrary.ndm under output directory: library not exists

... checking whether need to build cell library: sram_32_1024_freepdk45_TT_1p0V_25C_lib.ndm (2/2)
... checking whether can reuse library under output directory
cannot reuse CLIBs/sram_32_1024_freepdk45_TT_1p0V_25C_lib.ndm under output directory: library not exists

... processing cell library: NangateOpenCellLibrary.ndm (1/2)
... run lm_shell to build the cell library
...............................................................................
..............................................................................
................
created new cell library under output directory: CLIBs/NangateOpenCellLibrary.ndm

... processing cell library: sram_32_1024_freepdk45_TT_1p0V_25C_lib.ndm (2/2)
... run lm_shell to build the cell library
...................
created new cell library under output directory: CLIBs/sram_32_1024_freepdk45_TT_1p0V_25C_lib.ndm


Information: Successfully built 2 reference libraries: NangateOpenCellLibrary.ndm sram_32_1024_freepdk45_TT_1p0V_25C_lib.ndm. (LIB-093)
{raven_wrapperNangate}
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
RM-info : Reading full chip verilog (/home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v)
Information: Reading Verilog into new design 'raven_wrapper/init_dp' in library 'raven_wrapperNangate'. (VR-012)
Loading verilog file '/home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v'
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 22419 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 22456 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 22679 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 22731 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 22771 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 22823 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23185 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23231 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23277 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23323 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23369 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23415 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23467 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23507 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23964 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 23966 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '1' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24002 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24008 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '1' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24242 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '1' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24244 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24271 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '1' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24411 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24413 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24439 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24524 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24526 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24548 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24584 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '1' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24642 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24654 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24704 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24738 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24750 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '1' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24823 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24830 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24851 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24908 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24919 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24931 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 24992 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 25045 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '1' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 25097 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 25149 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 30600 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 37379 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 37968 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 37997 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38009 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38093 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38105 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38117 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38129 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38141 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38153 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38165 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38177 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38189 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38201 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38213 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38225 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38237 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38249 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38261 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38273 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38285 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38297 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38309 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38321 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 38333 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '00' requires 8 bits
        	which is too large for width 5. Truncated.
        	At line 139119 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 2. Truncated.
        	At line 139120 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 1. Truncated.
        	At line 139122 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '0' requires 4 bits
        	which is too large for width 3. Truncated.
        	At line 139123 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Warning: hex constant '00' requires 8 bits
        	which is too large for width 6. Truncated.
        	At line 139128 in /home/prakhan/task_5/icc2_workshop_collaterals/raven_wrapper.synth.v. (SVR-42)
Number of modules read: 1
Top level ports: 46
Total ports in all modules: 46
Total nets in all modules: 24294
Total instances in all modules: 21284
Elapsed = 00:00:00.10, CPU = 00:00:00.09
1
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
Warning: File './init_design.tech_setup.tcl' was not found in search path. (CMD-030)
RM-error : TCL_TECH_SETUP_FILE(./init_design.tech_setup.tcl) is invalid. Please correct it.
# Specify a Tcl script to read in your TLU+ files by using the read_parasitic_tech command
if {[file exists [which $TCL_PARASITIC_SETUP_FILE]]} {
   puts "RM-info : Sourcing [which $TCL_PARASITIC_SETUP_FILE]"
   source -echo $TCL_PARASITIC_SETUP_FILE
} elseif {$TCL_PARASITIC_SETUP_FILE != ""} {
   puts "RM-error : TCL_PARASITIC_SETUP_FILE($TCL_PARASITIC_SETUP_FILE) is invalid. Please correct it."
} else {
   puts "RM-info : No TLU plus files sourced, Parastic library containing TLU+ must be included in library reference list"
}
Warning: File './init_design.read_parasitic_tech_example.tcl' was not found in search path. (CMD-030)
RM-error : TCL_PARASITIC_SETUP_FILE(./init_design.read_parasitic_tech_example.tcl) is invalid. Please correct it.
###---Routing settings---###
## Set max routing layer
if {$MAX_ROUTING_LAYER != ""} {set_ignored_layers -max_routing_layer $MAX_ROUTING_LAYER}
Using libraries: raven_wrapperNangate NangateOpenCellLibrary sram_32_1024_freepdk45_TT_1p0V_25C_lib
Linking block raven_wrapperNangate:raven_wrapper/init_dp.design
Information: User units loaded from library 'NangateOpenCellLibrary' (LNK-040)
Design 'raven_wrapper' was successfully linked.
1
## Set min routing layer
if {$MIN_ROUTING_LAYER != ""} {set_ignored_layers -min_routing_layer $MIN_ROUTING_LAYER}
1
####################################
# Check Design: Pre-Floorplanning
####################################
if {$CHECK_DESIGN} {
   redirect -file ${REPORTS_DIR_INIT_DP}/check_design.pre_floorplan     {check_design -ems_database check_design.pre_floorplan.ems -checks dp_pre_floorplan}
}
[SCL] 12/21/2025 12:58:08 PID:12598 Client:nanodc.iitgn.ac.in Authorization failed ICCompilerII 2022.12
[SCL] 12/21/2025 12:58:08 PID:12598 Client:nanodc.iitgn.ac.in Authorization failed ICCompilerII-4 2022.12
[SCL] 12/21/2025 12:58:08 PID:12598 Client:nanodc.iitgn.ac.in Server:27020@c2s.cdacb.in Authorization succeeded ICCompilerII-8 2022.12
[SCL] 12/21/2025 12:58:09 PID:12598 Client:nanodc.iitgn.ac.in Server:27020@c2s.cdacb.in Authorization succeeded ICCompilerII-NX 2022.12
[SCL] 12/21/2025 12:58:09 Checking status for feature ICCompilerII-8
[SCL] 12/21/2025 12:58:09 PID:12598 Client:nanodc.iitgn.ac.in Server:27020@c2s.cdacb.in Checkout succeeded ICCompilerII-8 2022.12
[SCL] 12/21/2025 12:58:09 Checking status for feature ICCompilerII-NX
[SCL] 12/21/2025 12:58:09 PID:12598 Client:nanodc.iitgn.ac.in Server:27020@c2s.cdacb.in Checkout succeeded ICCompilerII-NX 2022.12
####################################
# Floorplanning (USER-DEFINED)
####################################
initialize_floorplan \
    -control_type die \
    -boundary {{0 0} {3588 5188}} \
    -core_offset {200 200 200 200}
[icc2-lic Sun Dec 21 12:58:09 2025] Command 'initialize_floorplan' requires licenses
[icc2-lic Sun Dec 21 12:58:09 2025] Attempting to check-out alternate set of keys directly with queueing
[icc2-lic Sun Dec 21 12:58:09 2025] Sending count request for 'ICCompilerII-8' 
[icc2-lic Sun Dec 21 12:58:09 2025] Count request for 'ICCompilerII-8' returned 1 
[icc2-lic Sun Dec 21 12:58:09 2025] Sending check-out request for 'ICCompilerII-8' (1) with wait option
[SCL] 12/21/2025 12:58:09 Checking status for feature ICCompilerII-8
[SCL] 12/21/2025 12:58:09 PID:12598 Client:nanodc.iitgn.ac.in Server:27020@c2s.cdacb.in Checkout succeeded ICCompilerII-8 2022.12
[icc2-lic Sun Dec 21 12:58:09 2025] Check-out request for 'ICCompilerII-8' with wait option succeeded
[icc2-lic Sun Dec 21 12:58:09 2025] Sending checkout check request for 'ICCompilerII-8' 
[icc2-lic Sun Dec 21 12:58:09 2025] Checkout check request for 'ICCompilerII-8' returned 0 
[icc2-lic Sun Dec 21 12:58:09 2025] Sending count request for 'ICCompilerII-8' 
[icc2-lic Sun Dec 21 12:58:09 2025] Count request for 'ICCompilerII-8' returned 1 
[icc2-lic Sun Dec 21 12:58:09 2025] Sending count request for 'ICCompilerII-NX' 
[icc2-lic Sun Dec 21 12:58:09 2025] Count request for 'ICCompilerII-NX' returned 1 
[icc2-lic Sun Dec 21 12:58:09 2025] Sending check-out request for 'ICCompilerII-NX' (1) with wait option
[SCL] 12/21/2025 12:58:09 Checking status for feature ICCompilerII-NX
[SCL] 12/21/2025 12:58:09 PID:12598 Client:nanodc.iitgn.ac.in Server:27020@c2s.cdacb.in Checkout succeeded ICCompilerII-NX 2022.12
[icc2-lic Sun Dec 21 12:58:09 2025] Check-out request for 'ICCompilerII-NX' with wait option succeeded
[icc2-lic Sun Dec 21 12:58:09 2025] Sending checkout check request for 'ICCompilerII-NX' 
[icc2-lic Sun Dec 21 12:58:09 2025] Checkout check request for 'ICCompilerII-NX' returned 0 
[icc2-lic Sun Dec 21 12:58:09 2025] Sending count request for 'ICCompilerII-NX' 
[icc2-lic Sun Dec 21 12:58:09 2025] Count request for 'ICCompilerII-NX' returned 1 
[icc2-lic Sun Dec 21 12:58:09 2025] Check-out of alternate set of keys directly with queueing was successful
Removing existing floorplan objects
Creating core...
Core utilization ratio = 0.52%
Unplacing all cells...
Creating site array...
Creating routing tracks...
Initializing floorplan completed.
save_block -force -label floorplan
Information: The command 'save_block' cleared the undo history. (UNDO-016)
Information: Saving 'raven_wrapperNangate:raven_wrapper/init_dp.design' to 'raven_wrapperNangate:raven_wrapper/floorplan.design'. (DES-028)
1
save_lib -all
Saving all libraries...
1
1
icc2_shell> gui_show_man_page
icc2_shell> get_ports
{pll_clk ext_clk ext_clk_sel ext_reset reset spi_sck {gpio[15]} {gpio[14]} {gpio[13]} {gpio[12]} {gpio[11]} {gpio[10]} {gpio[9]} {gpio[8]} {gpio[7]} {gpio[6]} {gpio[5]} {gpio[4]} {gpio[3]} {gpio[2]} {gpio[1]} {gpio[0]} analog_out_sel opamp_ena opamp_bias_ena comp_ena {comp_ninputsrc[1]} {comp_ninputsrc[0]} {comp_pinputsrc[1]} {comp_pinputsrc[0]} rcosc_ena overtemp_ena overtemp rcosc_in xtal_in comp_in ser_tx ser_rx irq_pin trap flash_csb flash_clk flash_io0 flash_io1 flash_io2 flash_io3}
icc2_shell> place_pins -self
[icc2-lic Sun Dec 21 13:01:24 2025] Command 'place_pins' requires licenses
[icc2-lic Sun Dec 21 13:01:24 2025] Attempting to check-out alternate set of keys directly with queueing
[icc2-lic Sun Dec 21 13:01:24 2025] Sending count request for 'ICCompilerII-8' 
[icc2-lic Sun Dec 21 13:01:24 2025] Count request for 'ICCompilerII-8' returned 1 
[icc2-lic Sun Dec 21 13:01:24 2025] Sending check-out request for 'ICCompilerII-8' (1) with wait option
[SCL] 12/21/2025 13:01:24 Checking status for feature ICCompilerII-8
[SCL] 12/21/2025 13:01:24 PID:12598 Client:nanodc.iitgn.ac.in Server:27020@c2s.cdacb.in Checkout succeeded ICCompilerII-8 2022.12
[icc2-lic Sun Dec 21 13:01:24 2025] Check-out request for 'ICCompilerII-8' with wait option succeeded
[icc2-lic Sun Dec 21 13:01:24 2025] Sending checkout check request for 'ICCompilerII-8' 
[icc2-lic Sun Dec 21 13:01:24 2025] Checkout check request for 'ICCompilerII-8' returned 0 
[icc2-lic Sun Dec 21 13:01:24 2025] Sending count request for 'ICCompilerII-8' 
[icc2-lic Sun Dec 21 13:01:24 2025] Count request for 'ICCompilerII-8' returned 1 
[icc2-lic Sun Dec 21 13:01:24 2025] Sending count request for 'ICCompilerII-NX' 
[icc2-lic Sun Dec 21 13:01:24 2025] Count request for 'ICCompilerII-NX' returned 1 
[icc2-lic Sun Dec 21 13:01:24 2025] Sending check-out request for 'ICCompilerII-NX' (1) with wait option
[SCL] 12/21/2025 13:01:24 Checking status for feature ICCompilerII-NX
[SCL] 12/21/2025 13:01:24 PID:12598 Client:nanodc.iitgn.ac.in Server:27020@c2s.cdacb.in Checkout succeeded ICCompilerII-NX 2022.12
[icc2-lic Sun Dec 21 13:01:24 2025] Check-out request for 'ICCompilerII-NX' with wait option succeeded
[icc2-lic Sun Dec 21 13:01:24 2025] Sending checkout check request for 'ICCompilerII-NX' 
[icc2-lic Sun Dec 21 13:01:24 2025] Checkout check request for 'ICCompilerII-NX' returned 0 
[icc2-lic Sun Dec 21 13:01:24 2025] Sending count request for 'ICCompilerII-NX' 
[icc2-lic Sun Dec 21 13:01:24 2025] Count request for 'ICCompilerII-NX' returned 1 
[icc2-lic Sun Dec 21 13:01:24 2025] Check-out of alternate set of keys directly with queueing was successful
Information: Starting 'place_pins' (FLW-8000)
Information: Time: 2025-12-21 13:01:24 / Session: 0.06 hr / Command: 0.00 hr / Memory: 581 MB (FLW-8100)
Load DB...
CPU Time for load db: 00:00:00.01u 00:00:00.00s 00:00:00.01e: 

Min routing layer: metal1
Max routing layer: metal10


CPU Time for Top Level Pre-Route Processing: 00:00:00.00u 00:00:00.00s 00:00:00.00e: 
Number of block ports: 30
Number of block pin locations assigned from router: 0
CPU Time for Pin Preparation: 00:00:00.00u 00:00:00.00s 00:00:00.00e: 
Number of PG ports on blocks: 0
Number of pins created: 30
CPU Time for Pin Creation: 00:00:00.00u 00:00:00.00s 00:00:00.00e: 
Total Pin Placement CPU Time: 00:00:00.10u 00:00:00.01s 00:00:00.11e: 
Information: Ending 'place_pins' (FLW-8001)
Information: Time: 2025-12-21 13:01:24 / Session: 0.06 hr / Command: 0.00 hr / Memory: 598 MB (FLW-8100)
1
icc2_shell> 
```
</details>



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
