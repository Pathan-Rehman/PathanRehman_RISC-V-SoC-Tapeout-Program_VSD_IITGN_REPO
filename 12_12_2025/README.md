# RISC-V Reference SoC Implementation using Synopsys and SCL180 PDK (RTL + Synthesis + GLS)

## Repository Structure
```
VsdRiscvScl180/
├── dv             # Contains functional verification files 
├── gl             # Contains GLS supports files
├── gls            # Contains test bench files and synthesized netlists
├── rtl            # Contains verilog files        
├── synthesis      # Contains synthesis scripts and outputs
   ├──output       # Contain synthesis output
   ├──report       # Contain area,power and qor post synth report
   ├──work         # Synthesis work folder
├── README.md      # This README file
```
## Prerequisites
Before using this repository, ensure you have the following dependencies installed:

- **SCL180 PDK** ( SCL180 PDK)
- **RiscV32-uknown-elf.gcc** (building functional simulation files)
- **Caravel User Project Framework** from Efabless
- **Synopsys EDA tool Suite** for Synthesis
- **Verilog Simulator** (e.g., Icarus Verilog)
- **GTKWAVE** (used for visualizing testbench waves)

## Test Instructions
### Repo Setup
1. Clone the repository:
   ```sh 
   git clone https://github.com/vsdip/vsdRiscvScl180.git
   cd vsdRiscvScl180
   git checkout iitgn
   ```
2. Install required dependencies (ensure dc_shell and SCL180 PDK are properly set up).

### Functional Simulation Setup
3. Setup functional simulation file paths
   - Edit Makefile at this path [./dv/hkspi/Makefile](./dv/hkspi/Makefile)
   - Modify and verify `GCC_Path` to point to correct riscv installation
   - Modify and verify `scl_io_PATH` to point to correct io

```
