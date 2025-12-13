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

1. Modify and verify following variables in synth.tcl file at path [./synthesis/synth.tcl](./synthesis/synth.tcl)
   ```
   library Path
   Root Directory Path
   SCL PDK Path
   SCL IO PATH

   ```
### Running Synthesis
9. open a terminal and cd to the work folder i.e. [./synthesis/work_folder](./synthesis/work_folder)
10. Run synthesis using following command
```
dc_shell -f ../synth.tcl
```

<img width="787" height="816" alt="image" src="https://github.com/user-attachments/assets/1f7ddfd0-c6f6-40c6-abf8-b137f6e28b33" />

This should update the caravel_snthesis.v file in [./synthesis/output](./synthesis/output) folder

<img width="679" height="79" alt="image" src="https://github.com/user-attachments/assets/8cfe6758-4e8b-4210-9011-dc41340da593" />

11. check out the reports

### Power Post Synth Report

<img width="786" height="752" alt="image" src="https://github.com/user-attachments/assets/54a7e6f6-2a76-45c3-9338-7f8ec02dc9d0" />

### QoR Post Synth Report

<img width="784" height="749" alt="image" src="https://github.com/user-attachments/assets/f0dc20c7-f62f-478e-ace0-b1284a798254" />

### Area Post Synth Report

<img width="786" height="751" alt="image" src="https://github.com/user-attachments/assets/467285bc-00cf-4de5-8637-7a6c7e64dd03" />

# GLS Setup


