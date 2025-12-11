# RTL AND GLS Simulation of HKSPI using Mixed (RTL + GLS) Technique

## Prerequisites

Open a GitHub Codespace (or Ubuntu terminal), ensure you have `git`, `iverilog`, and `python3` installed:

```bash
sudo apt-get update
sudo apt-get install -y git iverilog python3 python3-pip
```
<img width="858" height="662" alt="image" src="https://github.com/user-attachments/assets/dc527180-ed83-489e-bd85-d94f58def89a" />

***

## Step 1: Create workspace and clone Caravel

```bash
mkdir -p ~/caravel_vsd
cd ~/caravel_vsd
git clone https://github.com/efabless/caravel
cd caravel
```
<img width="769" height="269" alt="image" src="https://github.com/user-attachments/assets/8bf5a3e8-fa84-4ed2-a60c-8e7bdc489a03" />

***

## Step 2: Initialize git submodules

```bash
git submodule update --init --recursive
```
<img width="806" height="28" alt="image" src="https://github.com/user-attachments/assets/61276f9d-5f79-44f1-8380-a90dd3c160a7" />

***

## Step 3: Install volare and enable Sky130A PDK

```bash
pip install volare --user --upgrade
export PATH=$PATH:$HOME/.local/bin
export CARAVEL_ROOT=$(pwd)
export PDK_ROOT=$(pwd)/pdk
export PDK=sky130A
volare enable 12df12e2e74145e31c5a13de02f9a1e176b56e67
```

*This downloads the Sky130A PDK libraries into `$CARAVEL_ROOT/pdk/sky130A/`.*

<img width="1070" height="727" alt="image" src="https://github.com/user-attachments/assets/995026a9-fa68-4e94-b454-7921ee478de4" />
<img width="1081" height="714" alt="image" src="https://github.com/user-attachments/assets/ce69be58-cc69-40be-b349-ca1a45024dfd" />


***

## Step 4: Clone the management SoC wrapper

```bash
cd $CARAVEL_ROOT
rm -rf verilog/rtl/mgmt_core_wrapper
git clone https://github.com/efabless/caravel_mgmt_soc_litex verilog/rtl/mgmt_core_wrapper
```

*This brings in the LiteX management core (including VexRiscv) under `verilog/rtl/mgmt_core_wrapper/`.*

<img width="826" height="246" alt="image" src="https://github.com/user-attachments/assets/e08e4d01-b79a-4dc5-a3b1-d9312007f3f6" />


***

## Step 5: Patch `caravel_netlists.v` for GLS

```bash
cd $CARAVEL_ROOT/verilog/rtl
sed -i 's|"gl/digital_pll.v"|"digital_pll.v"|g' caravel_netlists.v
sed -i 's|"gl/gpio_control_block.v"|"gpio_control_block.v"|g' caravel_netlists.v
sed -i 's|"gl/gpio_signal_buffering.v"|"gpio_signal_buffering.v"|g' caravel_netlists.v
sed -i 's|"gl/mgmt_defines.v"|"defines.v"|g' caravel_netlists.v
sed -i 's|"gl/mgmt_core_wrapper.v"|"mgmt_core_wrapper.v"|g' caravel_netlists.v
sed -i 's|`include "housekeeping.v"|// `include "housekeeping.v"|g' caravel_netlists.v
```

*These fix broken `gl/` include paths and prevent duplicate RTL housekeeping from being pulled in during GLS.*

<img width="796" height="160" alt="image" src="https://github.com/user-attachments/assets/bae50fb3-021f-4ba1-bc05-cf1b8bc48c39" />


***

## Step 6: Create a dummy fill‑cell module

```bash
cd $CARAVEL_ROOT
echo 'module sky130_ef_sc_hd__fill_4(inout VPWR, inout VGND, inout VPB, inout VNB); endmodule' > verilog/dv/dummy_fill.v
```

*This stubs a filler cell used in GL netlists but not functionally needed.*

<img width="1091" height="69" alt="image" src="https://github.com/user-attachments/assets/f6344346-7218-4d06-8fa4-e1dfb2a7db2b" />

***

## Step 7: Navigate to the hkspi test directory

```bash
cd $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc/hkspi
```

***

## Step 8: (Optional) Disable VCD dumping to save disk space

```bash
sed -i 's/\$dumpfile/\/\/ \$dumpfile/g' hkspi_tb.v
sed -i 's/\$dumpvars/\/\/ \$dumpvars/g' hkspi_tb.v
```

*Comment out waveform generation for faster, lighter runs.*

<img width="1092" height="94" alt="image" src="https://github.com/user-attachments/assets/4d8fd339-e897-466c-afc1-6f8b80ea1c29" />


***

## Step 9: RTL simulation

### 9.1 Locate VexRiscv

```bash
export VEX_FILE=$(find $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper -name "VexRiscv*.v" | head -n 1)
```


### 9.2 Compile RTL with iverilog

```bash
iverilog -Ttyp \
  -DFUNCTIONAL -DSIM \
  -D USE_POWER_PINS \
  -D UNIT_DELAY=#1 \
  -I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
  -I $CARAVEL_ROOT/verilog/dv/caravel \
  -I $CARAVEL_ROOT/verilog/rtl \
  -I $CARAVEL_ROOT/verilog \
  -I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  -I $PDK_ROOT/sky130A \
  -y $CARAVEL_ROOT/verilog/rtl \
  -y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
  $CARAVEL_ROOT/verilog/dv/dummy_fill.v \
  $VEX_FILE \
  hkspi_tb.v -o hkspi_rtl.vvp
```


### 9.3 Run RTL simulation

```bash
vvp hkspi_rtl.vvp | tee rtl_hkspi.log
```

*You should see "Test HK SPI (RTL) Passed" in the log.*

<img width="1093" height="645" alt="image" src="https://github.com/user-attachments/assets/8de75048-97ae-4539-8d4c-bc393de9d361" />


***

## Step 10: GLS (mixed RTL+GL) simulation

### 10.1 Compile GLS with iverilog

```bash
iverilog -Ttyp \
  -DFUNCTIONAL -DSIM \
  -D USE_POWER_PINS \
  -D UNIT_DELAY=#1 \
  -I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
  -I $CARAVEL_ROOT/verilog/dv/caravel \
  -I $CARAVEL_ROOT/verilog/rtl \
  -I $CARAVEL_ROOT/verilog \
  -I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  -I $PDK_ROOT/sky130A \
  -y $CARAVEL_ROOT/verilog/rtl \
  -y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  $CARAVEL_ROOT/verilog/gl/housekeeping.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
  $CARAVEL_ROOT/verilog/dv/dummy_fill.v \
  $VEX_FILE \
  hkspi_tb.v -o hkspi_gls.vvp
```

*Here `verilog/gl/housekeeping.v` is the gate‑level netlist; other blocks remain RTL.*

### 10.2 Run GLS simulation

```bash
vvp hkspi_gls.vvp | tee gls_hkspi.log
```
<img width="1091" height="644" alt="image" src="https://github.com/user-attachments/assets/4714839f-e818-44bc-bd6d-5efc8912c20a" />

***

## Step 11: Compare RTL vs GLS results

### 11.1 Extract register reads

```bash
grep "Read register" rtl_hkspi.log > rtl_reads.txt
grep "Read register" gls_hkspi.log > gls_reads.txt
```


### 11.2 Diff the results

```bash
diff -s rtl_reads.txt gls_reads.txt
```

*If functional behavior matches, you'll see: **"Files rtl_reads.txt and gls_reads.txt are identical"**.*

<img width="1085" height="181" alt="image" src="https://github.com/user-attachments/assets/449f0b75-3e53-4aa7-8c55-6e810d44dd00" />


***

# errors

The error shows that `VexRiscv` module is not being found because you didn't include `$VEX_FILE` in your `iverilog` command. Looking at your command, the line breaks make it appear you may have accidentally omitted it.[1]

Here's the corrected command with `$VEX_FILE` properly included:

```bash
export VEX_FILE=$(find $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper -name "VexRiscv*.v" | head -n 1)

iverilog -Ttyp \
  -DFUNCTIONAL -DSIM \
  -D USE_POWER_PINS \
  -D UNIT_DELAY=#1 \
  -I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
  -I $CARAVEL_ROOT/verilog/dv/caravel \
  -I $CARAVEL_ROOT/verilog/rtl \
  -I $CARAVEL_ROOT/verilog \
  -I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  -I $PDK_ROOT/sky130A \
  -y $CARAVEL_ROOT/verilog/rtl \
  -y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
  $CARAVEL_ROOT/verilog/dv/dummy_fill.v \
  $VEX_FILE \
  hkspi_tb.v -o hkspi_rtl.vvp
```


## What's happening

- `mgmt_core.v` instantiates the `VexRiscv` CPU module at line 8426
- The `VexRiscv` module definition is in a separate file (`VexRiscv*.v`) inside the cloned `mgmt_core_wrapper` directory
- Without explicitly listing `$VEX_FILE` in the compile command, iverilog cannot find the module definition

## Verify VexRiscv file exists

Before running, double-check the file was found:

```bash
echo $VEX_FILE
ls -lh $VEX_FILE
```

You should see a path like:  
`/home/vscode/caravel_vsd/caravel/verilog/rtl/mgmt_core_wrapper/verilog/rtl/VexRiscv.v`

If `$VEX_FILE` is empty, it means the `mgmt_core_wrapper` clone in Step 4 didn't complete properly. Re-run:

```bash
cd $CARAVEL_ROOT
rm -rf verilog/rtl/mgmt_core_wrapper
git clone https://github.com/efabless/caravel_mgmt_soc_litex verilog/rtl/mgmt_core_wrapper
```

Then try the export and compile again.

# WHY GLS Passed is not showing

You're getting "Test HK SPI **(RTL) Passed**" in your GLS simulation because the testbench message is hardcoded in `hkspi_tb.v` and doesn't change based on compilation defines.

## Why this happens

The monitor statement in `hkspi_tb.v` likely looks something like:

```verilog
$display("Monitor: Test HK SPI (RTL) Passed");
```

This string is fixed at the Verilog source level—it doesn't check whether you compiled with `-DGL` or used gate-level netlists.

## How to confirm you're actually running GLS

Even though the message says "(RTL)", you **are** running a mixed RTL+GL simulation if:

1. **Your compile command included `$CARAVEL_ROOT/verilog/gl/housekeeping.v`** (the gate-level housekeeping netlist)
2. **The patched `caravel_netlists.v` commented out RTL housekeeping** to avoid duplicate modules

The functional behavior is what matters—the housekeeping SPI path is exercised through the GL netlist, backed by Sky130 standard cells.

## Verify you compiled GLS correctly

Check your compile command included the GL file:

```bash
grep "housekeeping.v" hkspi_gls.vvp
```

Or re-examine your iverilog command—it should have:

```bash
$CARAVEL_ROOT/verilog/gl/housekeeping.v \
```

not:

```bash
$CARAVEL_ROOT/verilog/rtl/housekeeping.v \
```


## Optional: Customize the testbench message

If you want the log to say "GLS Passed" for clarity, edit `hkspi_tb.v` before compiling GLS:

```bash
cp hkspi_tb.v hkspi_tb_gls.v
sed -i 's/Test HK SPI (RTL) Passed/Test HK SPI (GLS) Passed/g' hkspi_tb_gls.v
```

Then compile GLS using `hkspi_tb_gls.v` instead of `hkspi_tb.v`:

```bash
iverilog -Ttyp \
  -DFUNCTIONAL -DSIM \
  -D USE_POWER_PINS \
  -D UNIT_DELAY=#1 \
  -I $CARAVEL_ROOT/verilog/dv/caravel/mgmt_soc \
  -I $CARAVEL_ROOT/verilog/dv/caravel \
  -I $CARAVEL_ROOT/verilog/rtl \
  -I $CARAVEL_ROOT/verilog \
  -I $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  -I $PDK_ROOT/sky130A \
  -y $CARAVEL_ROOT/verilog/rtl \
  -y $CARAVEL_ROOT/verilog/rtl/mgmt_core_wrapper/verilog/rtl \
  $CARAVEL_ROOT/verilog/gl/housekeeping.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/primitives.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_sc_hd/verilog/sky130_fd_sc_hd.v \
  $PDK_ROOT/sky130A/libs.ref/sky130_fd_io/verilog/sky130_fd_io.v \
  $CARAVEL_ROOT/verilog/dv/dummy_fill.v \
  $VEX_FILE \
  hkspi_tb_gls.v -o hkspi_gls.vvp
```


## Bottom line

The message is cosmetic; the actual simulation **is** using GL housekeeping if you followed the compile steps correctly. The register read values matching between RTL and GLS logs proves functional equivalence, which is the real verification objective.[2][4][1]

