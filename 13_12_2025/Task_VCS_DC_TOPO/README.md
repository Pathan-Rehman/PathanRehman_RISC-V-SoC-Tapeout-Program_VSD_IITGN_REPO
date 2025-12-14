# RISC-V SoC Research Task - Synopsys VCS + DC_TOPO Flow (SCL180)

In this task I have replace all open-source simulation tools in the existing flow and transition fully to Synopsys tools, while preserving design correctness and documentation quality.

# Toolchain Migration

I completely remove the following tools in my flow:

   . iverilog
   . gtkwave

Tools I used in this lab are :

VCS - U-2023.03

<img width="733" height="113" alt="image" src="https://github.com/user-attachments/assets/7bad6c33-3228-4701-8dbb-7b90672122da" />

DC_TOPO - T-2022.03-SP5

<img width="376" height="143" alt="image" src="https://github.com/user-attachments/assets/c9b1a187-cf8c-46b5-9bb1-c29a1561f25b" />

# Functional simulation setup

## Prerequisites
Before using this repository, ensure you have the following dependencies installed:

- **SCL180 PDK** ( SCL180 PDK)
- **RiscV32-uknown-elf.gcc** (building functional simulation files)
- **Caravel User Project Framework** from vsd
- **Synopsys EDA tool Suite** for Synthesis

## Test Instructions
### Repo Setup
1. Clone the repository:
   ```sh 
   git clone https://github.com/vsdip/vsdRiscvScl180.git
   cd vsdRiscvScl180
   git checkout iitgn
   ```
2. Install required dependencies (ensure dc_shell and SCL180 PDK are properly set up).
3. source the synopsys tools
4. go to home
   ```
   csh
   source toolRC_iitgntapeout
   ```

### Functional Simulation Setup

5. Setup functional simulation file paths
   - Edit Makefile at this path [./dv/hkspi/Makefile](./dv/hkspi/Makefile)
   - Modify and verify `GCC_Path` to point to correct riscv installation
   - Modify and verify `scl_io_PATH` to point to correct io
   - edit the Makefile as follows
   
Here's the complete rewritten Makefile for Synopsys VCS:

```makefile
# SPDX-FileCopyrightText: 2020 Efabless Corporation
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# SPDX-License-Identifier: Apache-2.0

# removing pdk path as everything has been included in one whole directory for this example.
# PDK_PATH = $(PDK_ROOT)/$(PDK)
scl_io_PATH = "/home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/iopad/cio250/6M1L/verilog/tsl18cio250/zero"
VERILOG_PATH = ../../
RTL_PATH = $(VERILOG_PATH)/rtl
BEHAVIOURAL_MODELS = ../ 
RISCV_TYPE ?= rv32imc

FIRMWARE_PATH = ../
GCC_PATH?=/home/prakhan/rehman/riscv32-unknown-elf/bin
GCC_PREFIX?=riscv32-unknown-elf

SIM_DEFINES = +define+FUNCTIONAL +define+SIM

SIM?=RTL

.SUFFIXES:

PATTERN = hkspi

# Path to management SoC wrapper repository
scl_io_wrapper_PATH ?= $(RTL_PATH)/scl180_wrapper

# VCS compilation options
VCS_FLAGS = -sverilog +v2k -full64 -debug_all -lca -timescale=1ns/1ps
VCS_INCDIR = +incdir+$(BEHAVIOURAL_MODELS) \
             +incdir+$(RTL_PATH) \
             +incdir+$(scl_io_wrapper_PATH) \
             +incdir+$(scl_io_PATH)

# Output files
SIMV = simv
COMPILE_LOG = compile.log
SIM_LOG = simulation.log

.SUFFIXES:

all: compile

hex: ${PATTERN:=.hex}

# VCS Compilation target
compile: ${PATTERN}_tb.v ${PATTERN}.hex
	vcs $(VCS_FLAGS) $(SIM_DEFINES) $(VCS_INCDIR) \
	${PATTERN}_tb.v \
	-l $(COMPILE_LOG) \
	-o $(SIMV)

# Run simulation in batch mode
sim: compile
	./$(SIMV) -l $(SIM_LOG)

# Run simulation with GUI (DVE)
gui: compile
	./$(SIMV) -gui -l $(SIM_LOG) &

# Generate VPD waveform
vpd: compile
	./$(SIMV) -l $(SIM_LOG)
	@echo "VPD waveform generated. View with: dve -vpd vcdplus.vpd &"

# Generate FSDB waveform (if Verdi is available)
fsdb: compile
	./$(SIMV) -l $(SIM_LOG)
	@echo "FSDB waveform generated. View with: verdi -ssf <filename>.fsdb &"

#%.elf: %.c $(FIRMWARE_PATH)/sections.lds $(FIRMWARE_PATH)/start.s
#	${GCC_PATH}/${GCC_PREFIX}-gcc -march=$(RISCV_TYPE) -mabi=ilp32 -Wl,-Bstatic,-T,$(FIRMWARE_PATH)/sections.lds,--strip-debug -ffreestanding -nostdlib -o $@ $(FIRMWARE_PATH)/start.s $<

#%.hex: %.elf
#	${GCC_PATH}/${GCC_PREFIX}-objcopy -O verilog $< $@ 
	# to fix flash base address
#	sed -i 's/@10000000/@00000000/g' $@

#%.bin: %.elf
#	${GCC_PATH}/${GCC_PREFIX}-objcopy -O binary $< /dev/stdout | tail -c +1048577 > $@

check-env:
#ifndef PDK_ROOT
#	$(error PDK_ROOT is undefined, please export it before running make)
#endif
#ifeq (,$(wildcard $(PDK_ROOT)/$(PDK)))
#	$(error $(PDK_ROOT)/$(PDK) not found, please install pdk before running make)
#endif
ifeq (,$(wildcard $(GCC_PATH)/$(GCC_PREFIX)-gcc ))
	$(error $(GCC_PATH)/$(GCC_PREFIX)-gcc is not found, please export GCC_PATH and GCC_PREFIX before running make)
endif
# check for efabless style installation
ifeq (,$(wildcard $(PDK_ROOT)/$(PDK)/libs.ref/*/verilog))
#SIM_DEFINES := ${SIM_DEFINES} +define+EF_STYLE
endif

# ---- Clean ----

clean:
	rm -f $(SIMV) *.log *.vpd *.fsdb *.key
	rm -rf simv.daidir csrc DVEfiles verdiLog novas.* *.fsdb+
	rm -rf AN.DB

.PHONY: clean compile sim gui vpd fsdb all check-env

```

## Key Changes Made

1. **Replaced `iverilog` with `vcs`** compilation
2. **Changed `-I` to `+incdir+`** for include directories
3. **Changed `+define+` syntax** for SIM_DEFINES (VCS standard)
4. **Added VCS-specific flags**: `-sverilog`, `+v2k`, `-full64`, `-debug_all`, `-lca`
5. **Removed all `.vvp` and `.vcd` targets** - replaced with `compile`, `sim`, `gui`, `vpd`, `fsdb`
6. **Updated clean target** to remove VCS-generated files: `simv.daidir`, `csrc`, `DVEfiles`, etc.
7. **Added separate targets** for batch simulation and GUI simulation
8. **Completely removed** any reference to `iverilog`, `vvp`, or `gtkwave`

## Usage

- **Compile only**: `make compile`

<img width="776" height="914" alt="image" src="https://github.com/user-attachments/assets/a2ae8c0c-9331-4b8f-9a67-011fad0ca8a2" />

- **Compile and simulate**: `make sim`

<img width="792" height="731" alt="image" src="https://github.com/user-attachments/assets/a1a562d7-f077-41b6-8cba-e984d7de135b" />

- **Simulate with GUI**: `make gui` if verdi installed if not `gtkwave hkspi.vcd`
  
<img width="1586" height="867" alt="image" src="https://github.com/user-attachments/assets/1dc3782f-02dc-445e-ba06-4b9efa48ac4d" />

- **Clean build files**: `make clean`

## Errors you might encounter after changing Makefile follow the below solutions to clear those errors 

### Error 1: Variable TMPDIR (tmp) is selecting a non-existent directory.

<img width="732" height="240" alt="image" src="https://github.com/user-attachments/assets/80ddce4c-b859-4f5d-832a-9ce721472556" />


Create the tmp directory in your current location

```
mkdir -p tmp
```
- rerun the make compile command

#### Why This Happens

VCS needs a temporary directory to store intermediate compilation files. The setup script set TMPDIR=tmp, which is a relative path referring to a tmp subdirectory in your current working directory. When you're in the hkspi directory, VCS looks for hkspi/tmp, which doesn't exist.

### Error 2: Error-[IND] Identifier not declared

<img width="735" height="538" alt="image" src="https://github.com/user-attachments/assets/0c7daad6-9bdf-4b82-93e2-fa40523dac57" />

Now there's an error in dummy_schmittbuf.v where signals are not declared. This happens because default_nettype none is set somewhere in your code, which requires explicit declaration of all signals.

#### Fix the dummy_schmittbuf.v file

Step 1: Open the file

```
gedit ../../rtl/dummy_schmittbuf.v
```
Step 2: Change `default_nettype wire to `default_nettype none

```
`default_nettype wire

module dummy_schmittbuf (
    output UDP_OUT,
    input UDP_IN,
    input VPWR,
    input VGND
);
    
    assign UDP_OUT = UDP_IN;
    
endmodule

```
Step 3: In the same file change primitive name which contains a special character $ in dummy__udp_pwrgood_pp$PG module and referencing modules

from :
```
dummy__udp_pwrgood_pp$PG
```

to :
```
dummy__udp_pwrgood_pp_PG
```
Step 4: After all changes re-run the command make compile

<img width="776" height="914" alt="image" src="https://github.com/user-attachments/assets/b48e3bb4-f9bc-415f-893e-49df78c52d4b" />

# Synthesis Setup

    • Tool: Synopsys DC_TOPO
    • Synthesize vsdcaravel using SCL180 standard cell libraries
    • Constraints must be clean and reasonable
    • Output:
        ◦ Synthesized netlist
        ◦ Area, timing, and power reports
		
Important Constraint:

	1) POR and memory modules must remain as RTL modules
    2) Do not synthesize or replace them with macros yet
    3) Treat them as modules/blackboxes as appropriate

as per the task 2 we need to make modules like POR and memory modules such as RAM128 and RAM256 as blackbox and should not synthesize them. To do that we need to change synth.tcl with many changes. Follow the included synth.tcl file in the repository to make the modules as blackbox and synthesis rest of the logic.

```synth.tcl
# ========================================================================
# Synopsys DC Synthesis Script for vsdcaravel
# Modified to keep POR and Memory modules as complete RTL blackboxes
# ========================================================================

# ========================================================================
# Load technology libraries
# ========================================================================
read_db "/home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/iopad/cio250/4M1L/liberty/tsl18cio250_min.db"
read_db "/home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/stdcell/fs120/4M1IL/liberty/lib_flow_ff/tsl18fs120_scl_ff.db"

# ========================================================================
# Set library variables
# ========================================================================
set target_library "/home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/iopad/cio250/4M1L/liberty/tsl18cio250_min.db /home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/stdcell/fs120/4M1IL/liberty/lib_flow_ff/tsl18fs120_scl_ff.db"
set link_library "* /home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/iopad/cio250/4M1L/liberty/tsl18cio250_min.db /home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/stdcell/fs120/4M1IL/liberty/lib_flow_ff/tsl18fs120_scl_ff.db"
set_app_var target_library $target_library
set_app_var link_library $link_library

# ========================================================================
# Define directory paths
# ========================================================================
set root_dir "/home/prakhan/task_2/vsdRiscvScl180"
set io_lib "/home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/iopad/cio250/4M1L/verilog/tsl18cio250/zero"
set verilog_files "$root_dir/rtl"
set top_module "vsdcaravel"
set output_file "$root_dir/synthesis/output/vsdcaravel_synthesis.v"
set report_dir "$root_dir/synthesis/report"

# ========================================================================
# Configure Blackbox Handling
# ========================================================================
# Prevent automatic memory inference and template saving
set_app_var hdlin_infer_multibit default_none
set_app_var hdlin_auto_save_templates false
set_app_var compile_ultra_ungroup_dw false

# ========================================================================
# Create Blackbox Stub File for Memory and POR Modules
# ========================================================================
set blackbox_file "$root_dir/synthesis/memory_por_blackbox_stubs.v"
set fp [open $blackbox_file w]
puts $fp "// Blackbox definitions for memory and POR modules"
puts $fp "// Auto-generated by synthesis script"
puts $fp ""

# RAM128 blackbox
puts $fp "(* blackbox *)"
puts $fp "module RAM128(CLK, EN0, VGND, VPWR, A0, Di0, Do0, WE0);"
puts $fp "  input CLK, EN0, VGND, VPWR;"
puts $fp "  input \[6:0\] A0;"
puts $fp "  input \[31:0\] Di0;"
puts $fp "  input \[3:0\] WE0;"
puts $fp "  output \[31:0\] Do0;"
puts $fp "endmodule"
puts $fp ""

# RAM256 blackbox
puts $fp "(* blackbox *)"
puts $fp "module RAM256(VPWR, VGND, CLK, WE0, EN0, A0, Di0, Do0);"
puts $fp "  input CLK, EN0;"
puts $fp "  inout VPWR, VGND;"
puts $fp "  input \[7:0\] A0;"
puts $fp "  input \[31:0\] Di0;"
puts $fp "  input \[3:0\] WE0;"
puts $fp "  output \[31:0\] Do0;"
puts $fp "endmodule"
puts $fp ""

# dummy_por blackbox
puts $fp "(* blackbox *)"
puts $fp "module dummy_por(vdd3v3, vdd1v8, vss3v3, vss1v8, porb_h, porb_l, por_l);"
puts $fp "  inout vdd3v3, vdd1v8, vss3v3, vss1v8;"
puts $fp "  output porb_h, porb_l, por_l;"
puts $fp "endmodule"
puts $fp ""

close $fp
puts "INFO: Created blackbox stub file: $blackbox_file"

# ========================================================================
# Read RTL Files
# ========================================================================
# Read defines first
read_file $verilog_files/defines.v

# Read blackbox stubs FIRST (before actual RTL)
puts "INFO: Reading memory and POR blackbox stubs..."
read_file $blackbox_file -format verilog

# ========================================================================
# Read RTL files excluding memory and POR modules
# ========================================================================
puts "INFO: Building RTL file list (excluding RAM128.v, RAM256.v, and dummy_por.v)..."

# Get all verilog files
set all_rtl_files [glob -nocomplain ${verilog_files}/*.v]

# Define files to exclude
set exclude_files [list \
    "${verilog_files}/RAM128.v" \
    "${verilog_files}/RAM256.v" \
    "${verilog_files}/dummy_por.v" \
]

# Build list of files to read
set rtl_to_read [list]
foreach file $all_rtl_files {
    set excluded 0
    foreach excl_file $exclude_files {
        if {[string equal $file $excl_file]} {
            set excluded 1
            puts "INFO: Excluding $file (using blackbox instead)"
            break
        }
    }
    if {!$excluded} {
        lappend rtl_to_read $file
    }
}

puts "INFO: Reading [llength $rtl_to_read] RTL files..."

# Read all RTL files EXCEPT RAM128.v, RAM256.v, and dummy_por.v
read_file $rtl_to_read -define USE_POWER_PINS -format verilog

# ========================================================================
# Elaborate Design
# ========================================================================
puts "INFO: Elaborating design..."
elaborate $top_module

# ========================================================================
# Set Blackbox Attributes for Memory Modules
# ========================================================================
puts "INFO: Setting Blackbox Attributes for Memory Modules..."

# Mark RAM128 as blackbox
if {[sizeof_collection [get_designs -quiet RAM128]] > 0} {
    set_attribute [get_designs RAM128] is_black_box true -quiet
    set_dont_touch [get_designs RAM128]
    puts "INFO: RAM128 marked as blackbox"
}

# Mark RAM256 as blackbox
if {[sizeof_collection [get_designs -quiet RAM256]] > 0} {
    set_attribute [get_designs RAM256] is_black_box true -quiet
    set_dont_touch [get_designs RAM256]
    puts "INFO: RAM256 marked as blackbox"
}

# ========================================================================
# Set POR (Power-On-Reset) Module as Blackbox
# ========================================================================
puts "INFO: Setting POR module as blackbox..."

# Mark dummy_por as blackbox
if {[sizeof_collection [get_designs -quiet dummy_por]] > 0} {
    set_attribute [get_designs dummy_por] is_black_box true -quiet
    set_dont_touch [get_designs dummy_por]
    puts "INFO: dummy_por marked as blackbox"
}

# Handle any other POR-related modules (case insensitive)
foreach_in_collection por_design [get_designs -quiet "*por*"] {
    set design_name [get_object_name $por_design]
    if {![string equal $design_name "dummy_por"]} {
        set_dont_touch $por_design
        set_attribute $por_design is_black_box true -quiet
        puts "INFO: $design_name set as blackbox"
    }
}

# ========================================================================
# Protect blackbox instances from optimization
# ========================================================================
puts "INFO: Protecting blackbox instances from optimization..."

# Protect all instances of RAM128, RAM256, and dummy_por
foreach blackbox_ref {"RAM128" "RAM256" "dummy_por"} {
    set instances [get_cells -quiet -hierarchical -filter "ref_name == $blackbox_ref"]
    if {[sizeof_collection $instances] > 0} {
        set_dont_touch $instances
        set inst_count [sizeof_collection $instances]
        puts "INFO: Protected $inst_count instance(s) of $blackbox_ref"
    }
}

# ========================================================================
# Link Design
# ========================================================================
puts "INFO: Linking design..."
link

# ========================================================================
# Uniquify Design
# ========================================================================
puts "INFO: Uniquifying design..."
uniquify

# ========================================================================
# Read SDC constraints (if exists)
# ========================================================================
if {[file exists "$root_dir/synthesis/vsdcaravel.sdc"]} {
    puts "INFO: Reading timing constraints..."
    read_sdc "$root_dir/synthesis/vsdcaravel.sdc"
}

# ========================================================================
# Compile Design (Basic synthesis)
# ========================================================================
puts "INFO: Starting compilation..."
compile_ultra -incremental

# ========================================================================
# Write Outputs
# ========================================================================
puts "INFO: Writing output files..."

# Write Verilog netlist
write -format verilog -hierarchy -output $output_file
puts "INFO: Netlist written to: $output_file"

# Write DDC format for place-and-route
write -format ddc -hierarchy -output "$root_dir/synthesis/output/vsdcaravel_synthesis.ddc"
puts "INFO: DDC written to: $root_dir/synthesis/output/vsdcaravel_synthesis.ddc"

# Write SDC with actual timing constraints
write_sdc "$root_dir/synthesis/output/vsdcaravel_synthesis.sdc"
puts "INFO: SDC written to: $root_dir/synthesis/output/vsdcaravel_synthesis.sdc"

# ========================================================================
# Generate Reports
# ========================================================================
puts "INFO: Generating reports..."

report_area > "$report_dir/area.rpt"
report_power > "$report_dir/power.rpt"
report_timing -max_paths 10 > "$report_dir/timing.rpt"
report_constraint -all_violators > "$report_dir/constraints.rpt"
report_qor > "$report_dir/qor.rpt"

# Report on blackbox modules
puts "INFO: Generating blackbox module report..."
set bb_report [open "$report_dir/blackbox_modules.rpt" w]
puts $bb_report "========================================"
puts $bb_report "Blackbox Modules Report"
puts $bb_report "========================================"
puts $bb_report ""

foreach bb_module {"RAM128" "RAM256" "dummy_por"} {
    puts $bb_report "Module: $bb_module"
    set instances [get_cells -quiet -hierarchical -filter "ref_name == $bb_module"]
    if {[sizeof_collection $instances] > 0} {
        puts $bb_report "  Status: PRESENT"
        puts $bb_report "  Instances: [sizeof_collection $instances]"
        foreach_in_collection inst $instances {
            puts $bb_report "    - [get_object_name $inst]"
        }
    } else {
        puts $bb_report "  Status: NOT FOUND"
    }
    puts $bb_report ""
}
close $bb_report
puts "INFO: Blackbox report written to: $report_dir/blackbox_modules.rpt"

# ========================================================================
# Summary
# ========================================================================
puts ""
puts "INFO: ========================================"
puts "INFO: Synthesis Complete!"
puts "INFO: ========================================"
puts "INFO: Output netlist: $output_file"
puts "INFO: DDC file: $root_dir/synthesis/output/vsdcaravel_synthesis.ddc"
puts "INFO: SDC file: $root_dir/synthesis/output/vsdcaravel_synthesis.sdc"
puts "INFO: Reports directory: $report_dir"
puts "INFO: Blackbox stub file: $blackbox_file"
puts "INFO: "
puts "INFO: NOTE: The following modules are preserved as blackboxes:"
puts "INFO:   - RAM128 (Memory macro)"
puts "INFO:   - RAM256 (Memory macro)"
puts "INFO:   - dummy_por (Power-On-Reset circuit)"
puts "INFO: These modules will need to be replaced with actual macros during P&R"
puts "INFO: ========================================"

# Exit dc_shell
# dc_shell> exit

```
## What the synth.tcl script actually does?

Here's a comprehensive explanation of the complete synthesis TCL script, broken down block by block:

## 1. Load Technology Libraries
```tcl
read_db "/home/Synopsys/pdk/SCL_PDK_3/.../tsl18cio250_min.db"
read_db "/home/Synopsys/pdk/SCL_PDK_3/.../tsl18fs120_scl_ff.db"
```
**Purpose**: Load compiled Liberty (.db) files into DC memory
- **`read_db`**: Reads binary database files containing cell timing/power/area information
- **First file**: I/O pad library (cio250) for chip periphery
- **Second file**: Standard cell library (fs120) for core logic
- **`_min.db`**: Minimum timing corner (fast process, high voltage, low temp)
- **`_ff.db`**: Fast-fast corner for timing analysis

***

## 2. Set Library Variables
```tcl
set target_library "..."
set link_library "* ..."
set_app_var target_library $target_library
set_app_var link_library $link_library
```
**Purpose**: Configure which libraries DC will use for synthesis
- **`target_library`**: Libraries DC uses to map logic gates during synthesis (technology mapping)
- **`link_library`**: Libraries DC searches to resolve cell references (`*` = current design in memory)
- **`set_app_var`**: Sets Synopsys application variables (persistent across sessions)
- **Why both?**: `target_library` = synthesis destination, `link_library` = reference resolution

***

## 3. Define Directory Paths
```tcl
set root_dir "/home/prakhan/task_2/vsdRiscvScl180"
set io_lib "..."
set verilog_files "$root_dir/rtl"
set top_module "vsdcaravel"
set output_file "$root_dir/synthesis/output/vsdcaravel_synthesis.v"
set report_dir "$root_dir/synthesis/report"
```
**Purpose**: Create variables for file paths (easier maintenance)
- **`root_dir`**: Project root directory
- **`io_lib`**: I/O library Verilog models (not used in this script but defined)
- **`verilog_files`**: Source RTL directory
- **`top_module`**: Top-level design name for elaboration
- **`output_file`**: Where to write synthesized netlist
- **`report_dir`**: Where to write analysis reports

***

## 4. Configure Blackbox Handling
```tcl
set_app_var hdlin_infer_multibit default_none
set_app_var hdlin_auto_save_templates false
set_app_var compile_ultra_ungroup_dw false
```
**Purpose**: Control how DC handles HDL reading and memory inference
- **`hdlin_infer_multibit default_none`**: Prevents DC from automatically inferring arrays as memories (stops RAM synthesis)
- **`hdlin_auto_save_templates false`**: Disables automatic saving of memory templates
- **`compile_ultra_ungroup_dw false`**: Prevents ungrouping of DesignWare components (keeps hierarchy)

**Why needed**: Without these, DC would try to synthesize RAM128/RAM256 into flip-flops or SRAMs

***

## 5. Create Blackbox Stub File
```tcl
set blackbox_file "$root_dir/synthesis/memory_por_blackbox_stubs.v"
set fp [open $blackbox_file w]
puts $fp "// Blackbox definitions..."
puts $fp "(* blackbox *)"
puts $fp "module RAM128(...);"
# ... more module definitions
close $fp
```
**Purpose**: Dynamically create a Verilog file with empty module definitions
- **`open $blackbox_file w`**: Opens file for writing, returns file handle `$fp`
- **`puts $fp`**: Writes lines to file
- **`(* blackbox *)`**: Verilog attribute telling synthesis this is a blackbox
- **`close $fp`**: Closes file handle
- **Why?**: Provides module interfaces without implementation, preventing synthesis

**Key modules created**:
1. **RAM128**: 7-bit address (128 words), 32-bit data, 4-bit write enable
2. **RAM256**: 8-bit address (256 words), 32-bit data, 4-bit write enable  
3. **dummy_por**: Power-on-reset with power pins and reset outputs

***

## 6. Read RTL Files - Defines
```tcl
read_file $verilog_files/defines.v
```
**Purpose**: Read preprocessor defines first
- **`read_file`**: Reads Verilog source files
- **Why first?**: `defines.v` contains `define` macros used by other files (e.g., `USE_POWER_PINS`)

***

## 7. Read Blackbox Stubs
```tcl
read_file $blackbox_file -format verilog
```
**Purpose**: Read empty blackbox definitions BEFORE actual RTL
- **`-format verilog`**: Explicitly specify Verilog format
- **Why first?**: Ensures DC sees blackbox version before any actual implementation
- **Result**: DC knows RAM128, RAM256, dummy_por exist but won't synthesize them

***

## 8. Filter RTL Files
```tcl
set all_rtl_files [glob -nocomplain ${verilog_files}/*.v]
set exclude_files [list "${verilog_files}/RAM128.v" ...]
set rtl_to_read [list]
foreach file $all_rtl_files {
    set excluded 0
    foreach excl_file $exclude_files {
        if {[string equal $file $excl_file]} {
            set excluded 1
            break
        }
    }
    if {!$excluded} {
        lappend rtl_to_read $file
    }
}
```
**Purpose**: Build list of RTL files, excluding blackbox implementations
- **`glob -nocomplain`**: Get all `.v` files (no error if none found)
- **`list`**: Creates TCL list
- **`foreach`**: Loop through each file
- **`string equal`**: Compare file paths
- **`lappend`**: Append to list if not excluded
- **Why exclude?**: Prevents reading actual RAM/POR implementations that would override blackboxes

***

## 9. Read Filtered RTL
```tcl
read_file $rtl_to_read -define USE_POWER_PINS -format verilog
```
**Purpose**: Read all RTL except blackboxed modules
- **`-define USE_POWER_PINS`**: Sets Verilog preprocessor define (like `+define+USE_POWER_PINS`)
- **Reads**: All design files except RAM128.v, RAM256.v, dummy_por.v
- **Result**: DC has full design except blackbox internals

***

## 10. Elaborate Design
```tcl
elaborate $top_module
```
**Purpose**: Build design hierarchy and resolve module references
- **`elaborate`**: Processes RTL to create internal design database
- **What happens**:
  - Resolves module instantiations
  - Creates hierarchy tree
  - Infers sequential logic (registers)
  - Builds netlist connectivity
- **For blackboxes**: Uses stub definitions (port-only, no internals)

***

## 11. Set Blackbox Attributes - Memory
```tcl
if {[sizeof_collection [get_designs -quiet RAM128]] > 0} {
    set_attribute [get_designs RAM128] is_black_box true -quiet
    set_dont_touch [get_designs RAM128]
}
```
**Purpose**: Mark modules as blackboxes and protect from optimization
- **`get_designs -quiet RAM128`**: Query if RAM128 design exists (no error if missing)
- **`sizeof_collection`**: Returns number of objects in collection
- **`set_attribute ... is_black_box true`**: Tells DC this is a blackbox (don't optimize internals)
- **`set_dont_touch`**: Prevents DC from:
  - Optimizing/removing the module
  - Modifying its ports
  - Ungrouping its hierarchy
- **`-quiet`**: Suppresses warnings

**Same for RAM256 and dummy_por**

***

## 12. Protect POR Modules (Wildcard)
```tcl
foreach_in_collection por_design [get_designs -quiet "*por*"] {
    set design_name [get_object_name $por_design]
    if {![string equal $design_name "dummy_por"]} {
        set_dont_touch $por_design
        set_attribute $por_design is_black_box true -quiet
    }
}
```
**Purpose**: Find and protect any other POR-related modules
- **`get_designs "*por*"`**: Wildcard search for modules with "por" in name
- **`foreach_in_collection`**: Loop through collection
- **`get_object_name`**: Extract name from design object
- **Why skip dummy_por?**: Already handled above (avoid redundancy)
- **Safety net**: Catches any POR modules with different naming

***

## 13. Protect Blackbox Instances
```tcl
foreach blackbox_ref {"RAM128" "RAM256" "dummy_por"} {
    set instances [get_cells -quiet -hierarchical -filter "ref_name == $blackbox_ref"]
    if {[sizeof_collection $instances] > 0} {
        set_dont_touch $instances
    }
}
```
**Purpose**: Protect instances (not just module definitions)
- **`get_cells`**: Query cell instances in design
- **`-hierarchical`**: Search entire hierarchy (not just top level)
- **`-filter "ref_name == $blackbox_ref"`**: Find instances of specific module type
- **`set_dont_touch $instances`**: Protect instances from:
  - Being optimized away
  - Having logic pushed through them
  - Boundary optimization
- **Why needed?**: Module protection doesn't automatically protect instances

***

## 14. Link Design
```tcl
link
```
**Purpose**: Resolve all module/cell references and check for unlinked objects
- **What it does**:
  - Connects module instances to definitions
  - Resolves library cell references
  - Checks for missing definitions
- **Errors if**:
  - Referenced module not found
  - Required library cell missing
- **For blackboxes**: Links instances to blackbox definitions (not internals)

***

## 15. Uniquify Design
```tcl
uniquify
```
**Purpose**: Create unique copies of multiply-instantiated modules
- **Why needed**: If module is instantiated multiple times, DC creates unique copies for independent optimization
- **Example**: If `moduleA` is used twice:
  - Before: `moduleA`, `moduleA`
  - After: `moduleA_0`, `moduleA_1`
- **For blackboxes**: They remain unchanged (protected by `dont_touch`)

***

## 16. Read Timing Constraints
```tcl
if {[file exists "$root_dir/synthesis/vsdcaravel.sdc"]} {
    read_sdc "$root_dir/synthesis/vsdcaravel.sdc"
}
```
**Purpose**: Load timing constraints (clock definitions, I/O delays, false paths)
- **`file exists`**: Check if SDC file exists
- **`read_sdc`**: Parses Synopsys Design Constraints file
- **SDC contains**:
  - Clock definitions (`create_clock`)
  - Input/output delays (`set_input_delay`)
  - Timing exceptions (`set_false_path`)
  - Max/min delays
- **Why conditional?**: Script won't fail if SDC missing

***

## 17. Compile Design
```tcl
compile
```
**Purpose**: Perform logic synthesis (main synthesis step)
- **What happens**:
  1. **Logic optimization**: Simplifies Boolean equations
  2. **Technology mapping**: Maps gates to standard cells from `target_library`
  3. **Timing optimization**: Meets setup/hold timing
  4. **Area optimization**: Minimizes cell area
  5. **DFT insertion** (if configured): Adds scan chains
- **For blackboxes**: Instances treated as black boxes (no internal optimization)
- **Output**: Gate-level netlist using library cells

**Alternative**: `compile_ultra` (more aggressive, longer runtime)

***

## 18. Write Outputs - Verilog Netlist
```tcl
write -format verilog -hierarchy -output $output_file
```
**Purpose**: Save synthesized netlist as Verilog
- **`-format verilog`**: Output format (alternatives: `ddc`, `vhdl`)
- **`-hierarchy`**: Include full hierarchy (not flattened)
- **`-output`**: Destination file
- **Contains**:
  - Module definitions
  - Instantiated standard cells
  - Wire connections
  - Blackbox module definitions (port-only)

***

## 19. Write Outputs - DDC
```tcl
write -format ddc -hierarchy -output ".../vsdcaravel_synthesis.ddc"
```
**Purpose**: Save design in Synopsys binary format
- **`ddc`**: Synopsys proprietary binary format
- **Advantages**:
  - Preserves all DC internal information
  - Faster to read than Verilog
  - Used by ICC/ICC2 for place-and-route
- **Contains**: Complete design database (netlist, constraints, attributes)

***

## 20. Write Outputs - SDC
```tcl
write_sdc "$root_dir/synthesis/output/vsdcaravel_synthesis.sdc"
```
**Purpose**: Export timing constraints with actual netlist names
- **Difference from input SDC**:
  - Input: Uses RTL signal names
  - Output: Uses gate-level netlist names
- **Used by**: P&R tools for timing closure
- **Contains**: Clock definitions, I/O constraints mapped to synthesized netlist

***

## 21. Generate Reports
```tcl
report_area > "$report_dir/area.rpt"
report_power > "$report_dir/power.rpt"
report_timing -max_paths 10 > "$report_dir/timing.rpt"
report_constraint -all_violators > "$report_dir/constraints.rpt"
report_qor > "$report_dir/qor.rpt"
```
**Purpose**: Generate analysis reports for design evaluation

### Individual Reports:
- **`report_area`**: Cell area, combinational vs sequential breakdown
- **`report_power`**: Power consumption (dynamic, static, switching)
- **`report_timing -max_paths 10`**: 10 worst timing paths (setup/hold analysis)
- **`report_constraint -all_violators`**: Lists all violated timing constraints
- **`report_qor`**: Quality of Results summary (area, timing, power overview)

***

## 22. Generate Blackbox Report
```tcl
set bb_report [open "$report_dir/blackbox_modules.rpt" w]
foreach bb_module {"RAM128" "RAM256" "dummy_por"} {
    set instances [get_cells -quiet -hierarchical -filter "ref_name == $bb_module"]
    if {[sizeof_collection $instances] > 0} {
        puts $bb_report "  Status: PRESENT"
        puts $bb_report "  Instances: [sizeof_collection $instances]"
        foreach_in_collection inst $instances {
            puts $bb_report "    - [get_object_name $inst]"
        }
    }
}
close $bb_report
```
**Purpose**: Create custom report documenting blackbox modules
- **Opens file**: For writing
- **Loops through**: Each blackbox module
- **Queries**: Instance count and names
- **Reports**:
  - Whether module is present
  - How many instances
  - Instance hierarchical paths
- **Result**: Verification that blackboxes are preserved

***

## 23. Summary
```tcl
puts ""
puts "INFO: ========================================"
puts "INFO: Synthesis Complete!"
# ... prints summary information
```
**Purpose**: Display synthesis completion message and file locations
- **`puts`**: Print to console (stdout)
- **Shows**:
  - Output file locations
  - Blackbox modules preserved
  - Next steps (P&R)
- **User-friendly**: Clear summary of synthesis results

***

<img width="1148" height="745" alt="image" src="https://github.com/user-attachments/assets/3e6f4e2e-8dde-4ca5-a4a6-690ff0c74754" />


This script ensures RAM128, RAM256, and dummy_por remain as port-only blackboxes, ready for macro replacement during physical design!

## Running tcl file

- To run the tcl file change to the synthesis directory and run the following command
  ```
	dc_shell -f synth.tcl | tee synthesis_complete.log
  ```
- The above command will do synthesis using dc_shell tool and create a log file.

<img width="738" height="662" alt="image" src="https://github.com/user-attachments/assets/f205ea8e-ece9-4968-bf07-7f60a4a2e18b" />

- After synthesis go to output directory and check the `vsdcaravel_synthesis.v`. The POR and Memory modules will have only ports.

<img width="791" height="748" alt="image" src="https://github.com/user-attachments/assets/9edfa89b-128f-4a53-9574-fe0cd48dbf02" />

## Reports

After synthesis report files will be generated in the report directory such as area, power, timing, constraints, qor and blackbox_modules reports.

<img width="730" height="276" alt="image" src="https://github.com/user-attachments/assets/f76d19ad-03ee-4add-9475-e44451550618" />


### area report

<img width="793" height="214" alt="image" src="https://github.com/user-attachments/assets/cf3e67c2-5c26-4f90-8a41-0c35211cc8bf" />
<img width="675" height="378" alt="image" src="https://github.com/user-attachments/assets/e0d25b1a-9934-49d0-a713-525dfc113762" />

### power report

<img width="789" height="501" alt="image" src="https://github.com/user-attachments/assets/b902c57d-ac75-49b1-bcc1-73c1ac7dc8a9" />
<img width="1152" height="746" alt="image" src="https://github.com/user-attachments/assets/6accb5ce-89b4-4a86-a0d5-e2ac9d4ebaa8" />

### qor report

<img width="1154" height="193" alt="image" src="https://github.com/user-attachments/assets/2607ea3b-a48e-4e48-a645-413d3efc4cb0" />
<img width="673" height="373" alt="image" src="https://github.com/user-attachments/assets/9fc97443-332a-4106-9a22-982eb4b6718d" />

### timing report

<img width="1152" height="320" alt="image" src="https://github.com/user-attachments/assets/90d08744-5fa4-4ce8-93e9-32a97bd24e28" />
<img width="719" height="589" alt="image" src="https://github.com/user-attachments/assets/afde1417-ac05-487c-a336-f30967418b5b" />

### constraints

<img width="1151" height="251" alt="image" src="https://github.com/user-attachments/assets/22b8ff3e-3256-4b8c-b8ab-e84737d1cf27" />

### Blackbox_modules

<img width="721" height="390" alt="image" src="https://github.com/user-attachments/assets/5021ffe3-27ca-4088-9268-6ed726b5a9e3" />

# GLS Setup

- As we made 3 modules as blackbox we need to add it's RTL Functionality to pass the GL Test, so we need to include those files in caravel_netlist.v
- Copy all rtl files into gl directory, then edit the Makefile as per below.

```
# SPDX-FileCopyrightText: 2020 Efabless Corporation
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# SPDX-License-Identifier: Apache-2.0

scl_io_PATH = "/home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/iopad/cio250/4M1L/verilog/tsl18cio250/zero"
scl_io_wrapper_PATH = ../rtl/scl180_wrapper
VERILOG_PATH = ../
RTL_PATH = $(VERILOG_PATH)/rtl
GL_PATH = $(VERILOG_PATH)/gl
BEHAVIOURAL_MODELS = ../gls
RISCV_TYPE ?= rv32imc
PDK_PATH = /home/Synopsys/pdk/SCL_PDK_3/SCLPDK_V3.0_KIT/scl180/stdcell/fs120/4M1IL/verilog/vcs_sim_model 
FIRMWARE_PATH = ../gls
GCC_PATH?=/home/prakhan/rehman/riscv32-unknown-elf/bin
GCC_PREFIX?=riscv32-unknown-elf

SIM_DEFINES = +define+FUNCTIONAL +define+SIM
SIM?=gl

.SUFFIXES:

PATTERN = hkspi

all: ${PATTERN:=.vcd}
hex: ${PATTERN:=.hex}
vcd: ${PATTERN:=.vcd}

# VCS compilation target
simv: ${PATTERN}_tb.v ${PATTERN}.hex
	 vcs -full64 -debug_access+all \
	 $(SIM_DEFINES) +define+GL \
	 -timescale=1ns/1ps \
	 +v2k -sverilog \
	 -lca -kdb \
	 +incdir+$(VERILOG_PATH) \
	 +incdir+$(VERILOG_PATH)/synthesis/output \
	 +incdir+$(BEHAVIOURAL_MODELS) \
	 +incdir+$(RTL_PATH) \
	 +incdir+$(GL_PATH) \
	 +incdir+$(scl_io_wrapper_PATH) \
	 +incdir+$(scl_io_PATH) \
	 +incdir+$(PDK_PATH) \
	 -y $(scl_io_wrapper_PATH) +libext+.v+.sv \
	 -y $(RTL_PATH) +libext+.v+.sv \
	 -y $(GL_PATH) +libext+.v+.sv \
	 -y $(scl_io_PATH) +libext+.v+.sv \
	 -y $(PDK_PATH) +libext+.v+.sv \
	 $(GL_PATH)/defines.v \
	 $< \
	 -l vcs_compile.log \
	 -o simv

# Run simulation and generate VCD
%.vcd: simv
	 ./simv +vcs+dumpvars+${PATTERN}.vcd \
	 -l simulation.log

# Alternative: Generate FSDB waveform (if Verdi is available)
%.fsdb: simv
	 ./simv -ucli -do "dump -file ${PATTERN}.fsdb -type fsdb -add {*}" \
	 -l simulation.log

%.elf: %.c $(FIRMWARE_PATH)/sections.lds $(FIRMWARE_PATH)/start.s
	 ${GCC_PATH}/${GCC_PREFIX}-gcc -march=$(RISCV_TYPE) -mabi=ilp32 -Wl,-Bstatic,-T,$(FIRMWARE_PATH)/sections.lds,--strip-debug -ffreestanding -nostdlib -o $@ $(FIRMWARE_PATH)/start.s $<

%.hex: %.elf
	 ${GCC_PATH}/${GCC_PREFIX}-objcopy -O verilog $< $@ 
 # to fix flash base address
	 sed -i 's/@10000000/@00000000/g' $@

%.bin: %.elf
	 ${GCC_PATH}/${GCC_PREFIX}-objcopy -O binary $< /dev/stdout | tail -c +1048577 > $@

# Interactive debug with DVE
debug: simv
	 ./simv -gui -l simulation.log

# Coverage report generation (optional)
coverage: simv
	 ./simv -cm line+cond+fsm+tgl -cm_dir coverage.vdb
	 urg -dir coverage.vdb -report urgReport

check-env:
ifeq (,$(wildcard $(GCC_PATH)/$(GCC_PREFIX)-gcc ))
	 $(error $(GCC_PATH)/$(GCC_PREFIX)-gcc is not found, please export GCC_PATH and GCC_PREFIX before running make)
endif

clean:
	 rm -f *.elf *.hex *.bin *.vcd *.fsdb *.log simv
	 rm -rf csrc simv.daidir DVEfiles ucli.key *.vpd urgReport coverage.vdb AN.DB

.PHONY: clean hex vcd fsdb all debug coverage check-env# SPDX-FileCopyrightText: 2020 Efabless Corporation


```
then run the following commands

- make clean
- Then copy the hkspi.hex file from dv/hkspi
- Then run `make`
<img width="834" height="802" alt="image" src="https://github.com/user-attachments/assets/21b1f9ee-d004-4eb5-a6f1-9493bd87f120" />
<img width="681" height="474" alt="image" src="https://github.com/user-attachments/assets/e49de8ac-7ab8-41a2-9db3-11db060da455" />

- To view the waveform use vcd file and open it using gtkwave

<img width="1668" height="964" alt="image" src="https://github.com/user-attachments/assets/e7dc194a-9cc5-4e58-aab1-1cb299930546" />

Synopsys Solvnet References

<img width="1910" height="602" alt="image" src="https://github.com/user-attachments/assets/750524ef-280c-43a7-975d-ef0f4e79b6b1" />

- To perform the synthesis in topological method we need to use `compile_ultra -incremental` instead of `compile` in tcl file, this do the synthesis in topological method.

