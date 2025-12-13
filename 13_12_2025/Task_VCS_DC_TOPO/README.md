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
   

