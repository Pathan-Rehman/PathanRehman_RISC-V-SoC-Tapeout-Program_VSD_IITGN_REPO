# POR Usage Analysis - VSD RISC-V SoC (SCL-180)

**Date:** December 15, 2025, 11:36 AM IST  
**Author:** Prakhan  
**Repository:** vsdRiscvScl180  
**Branch:** task3-por-removal  
**Status:** COMPLETE ANALYSIS

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [POR Module Implementation](#por-module-implementation)
3. [Signal Definitions](#signal-definitions)
4. [Detailed Usage Analysis by File](#detailed-usage-analysis-by-file)
5. [Reset Distribution Architecture](#reset-distribution-architecture)
6. [Dependency Mapping](#dependency-mapping)
7. [Critical Findings](#critical-findings)
8. [Removal Strategy](#removal-strategy)

---

## Executive Summary

The VSD Caravel-based RISC-V SoC uses a behavioral `dummy_por` module to generate three power-on reset signals (`porb_h`, `porb_l`, `por_l`) that distribute reset to various system components. This analysis identifies:

- ✅ **Where POR is used**: 8 files with 20+ usage locations
- ✅ **What it drives**: I/O pads, housekeeping logic, clock control, core reset
- ✅ **Which blocks depend on it**: 5 modules with actual POR dependency
- ✅ **Which are pass-through only**: 1 module (mgmt_core) - can be removed entirely
- ✅ **Dead code**: Pad enable signals that don't connect to SCL-180 pads

**Key Finding:** With SCL-180 pads (no ENABLE pins), only housekeeping and clock control have real POR dependencies, and both can use external reset instead.

---

## POR Module Implementation

### Location and Structure

**Primary Files:**
- `./rtl/dummy_por.v` (87 lines)
- `./rtl/simple_por.v` (87 lines - identical)
- `./gl/dummy_por.v` (gate-level version)

### Module Interface

```verilog
module dummy_por(
`ifdef USE_POWER_PINS
    inout vdd3v3,      // 3.3V power supply
    inout vdd1v8,      // 1.8V power supply
    inout vss3v3,      // 3.3V ground
    inout vss1v8,      // 1.8V ground
`endif
    output porb_h,     // Power-on reset bar (3.3V domain)
    output porb_l,     // Power-on reset bar (1.8V domain)
    output por_l       // Power-on reset (1.8V domain, active high)
);
```

### Implementation Details

**Simulation-Only Behavioral Model:**
```verilog
`ifdef SIM
    // 500ns delay (actual circuit: 15ms)
    `ifdef USE_POWER_PINS
        always @(posedge vdd3v3) begin
            #500 inode <= 1'b1;
        end
    `endif
    
    // Two schmitt triggers for hysteresis
    dummy__schmittbuf_1 hystbuf1 (.A(inode), .X(mid));
    dummy__schmittbuf_1 hystbuf2 (.A(mid), .X(porb_h));
    
    // SCL-180 pads have level-shifters already available
    assign porb_l = porb_h;
    assign por_l = ~porb_l;
`endif
```

**Critical Comment in Code (Line 81):**
```verilog
// since SCL180 has level-shifters already available in I/O pads
assign porb_l = porb_h;
```

---

## Signal Definitions

### porb_h (Power-On Reset Bar - High Voltage)

| Property | Value |
|----------|-------|
| **Voltage Domain** | 3.3V (I/O domain) |
| **Active State** | Active Low (bar = inverted) |
| **Source** | dummy_por module (from schmitt trigger chain) |
| **Typical Behavior** | Held LOW during power-up (~500ns), released HIGH when power stable |

**What porb_h Drives:**
1. **chip_io.v line 115** - `mprj_io_enh` (pad enable signals for ALL multi-project I/O)
   ```verilog
   assign mprj_io_enh = {`MPRJ_IO_PADS{porb_h}};
   ```
   **Status:** ⚠️ **DEAD CODE** - SCL-180 pads don't have ENABLE pins

2. **chip_io.v line 1121** - Reset pad `ENABLE_H` (SKY130 pad - COMMENTED OUT)
   ```verilog
   .ENABLE_H(porb_h),  // Power-on-reset
   ```
   **Status:** ❌ **UNUSED** - SKY130 code, replaced with SCL-180 simple input buffer

3. **mprj_io.v line 38** - Input to multi-project I/O module
   ```verilog
   input porb_h,
   ```

4. **mgmt_core.v** - Pass-through to `porb_h_out`
   ```verilog
   input wire porb_h_in,
   output wire porb_h_out
   assign porb_h_out = porb_h_in;
   ```
   **Status:** ⚠️ **NO LOGIC** - Just wiring, can be removed

---

### porb_l (Power-On Reset Bar - Low Voltage)

| Property | Value |
|----------|-------|
| **Voltage Domain** | 1.8V (Core domain) |
| **Active State** | Active Low (bar = inverted) |
| **Source** | Direct assignment from `porb_h` (no separate level shifter logic needed - comment confirms SCL-180 has built-in) |
| **Derivation** | `assign porb_l = porb_h;` (line 81 of dummy_por.v) |

**What porb_l Drives:**
1. **caravel_core.v line 528** - Housekeeping module input
   ```verilog
   .porb(porb_l),
   ```

2. **caravel_core.v line 591** - Clock control module input
   ```verilog
   .porb(porb_l),
   ```

3. **housekeeping.v line 82** - Input port declaration
   ```verilog
   input porb,
   ```

---

### por_l (Power-On Reset - Low Voltage, Active High)

| Property | Value |
|----------|-------|
| **Voltage Domain** | 1.8V (Core domain) |
| **Active State** | Active HIGH (inverted from porb_l) |
| **Source** | Inverted from `porb_l` (assign `por_l = ~porb_l;`) |
| **Polarity** | Opposite of porb_l - HIGH means reset released |

**What por_l Drives:**
1. **vsdcaravel.v line 255** - caravel_core module
   ```verilog
   .por(por_l),
   ```

2. **vsdcaravel.v line 313** - mgmt_core_wrapper module
   ```verilog
   .por_l(por_l),
   ```

---

## Detailed Usage Analysis by File

### 1. vsdcaravel.v (Top-Level Module)

**Lines with POR:** 175-177, 254-255, 312-313

#### Signal Declarations (Lines 175-177)
```verilog
// Power-on-reset signal.  The reset pad generates the sense-inverted
// reset at 3.3V.  The 1.8V signal and the inverted 1.8V signal are
// derived.

wire porb_h;
wire porb_l;
wire por_l;
```

**Purpose:** Declare internal wires for POR signals generated by caravel_core

**Dependency:** ✅ **Critical** - These signals come from dummy_por and distribute throughout design

#### caravel_core Connections (Lines 254-255)
```verilog
.porb_h(porb_h),
.por(por_l),
```

**Purpose:** Connect POR signals from caravel_core module to top-level

**Dependency:** ✅ **Critical** - Interface between caravel_core and top-level hierarchy

#### mgmt_core_wrapper Connections (Lines 312-313)
```verilog
.porb_h(porb_h),
.por_l(por_l),
```

**Purpose:** Route POR signals to management core module for pass-through

**Dependency:** ⚠️ **Pass-through only** - No actual logic uses these in mgmt_core

---

### 2. caravel_core.v (Core Logic Module)

**Lines with POR:** 60-61, 1382-1391, 528, 591

#### Output Declarations (Lines 60-61)
```verilog
output porb_h,
output por_l,
```

**Purpose:** Declare POR signals as outputs from caravel_core

**Dependency:** ✅ **Critical** - Generated by dummy_por instantiation, distributed to entire chip

#### dummy_por Instantiation (Lines 1382-1391) - THE MAIN POR SOURCE
```verilog
// Power-on-reset circuit
dummy_por por (
    `ifdef USE_POWER_PINS
        .vdd3v3(vddio),
        .vdd1v8(vccd),
        .vss3v3(vssio),
        .vss1v8(vssd),
    `endif
        .porb_h(porb_h),
        .porb_l(porb_l),
        .por_l(por_l)
);
```

**Purpose:** **MAIN SOURCE OF ALL POR SIGNALS** - instantiates behavioral dummy_por module

**Dependency:** ✅ **CRITICAL** - Without this, no POR signals exist

**Status:** ⚠️ **Non-synthesizable** - all logic is in `ifdef SIM` block

#### Housekeeping Connection (Line 528)
```verilog
.porb(porb_l),
```

**Purpose:** Connect porb_l to housekeeping module for reset distribution

**Dependency:** ✅ **Critical** - Controls housekeeping reset behavior

#### Clock Control Connection (Line 591)
```verilog
.porb(porb_l),
```

**Purpose:** Connect porb_l to clock control module for reset sequencing

**Dependency:** ✅ **Critical** - Controls clock output and reset logic

---

### 3. housekeeping.v (Housekeeping Controller)

**Lines with POR:** 82, 265, 267, 756, 921-922, 1032-1033

#### Input Port (Line 82)
```verilog
input porb,    // Primary reset
```

**Purpose:** Receive porb_l from caravel_core

**Dependency:** ✅ **Critical** - Source of reset for housekeeping logic

#### Flash SPI Output Control (Lines 265, 267)
```verilog
assign pad_flash_csb_oeb = (pass_thru_mgmt_delay) ? 1'b0 : (~porb ? 1'b1 : 1'b0);
assign pad_flash_clk_oeb = (pass_thru_mgmt) ? 1'b0 : (~porb ? 1'b1 : 1'b0);
```

**Purpose:** Disable flash SPI output drivers when porb is LOW (during reset)

**Logic:**
- When `porb = 0` (reset asserted) → `~porb = 1` → output enable = 1 (DISABLE output)
- When `porb = 1` (reset released) → `~porb = 0` → output enable follows SPI logic

**Dependency:** ✅ **Required** - Ensures flash pins are tristated during reset

**Replacement:** Can use external `resetb` signal directly

#### Housekeeping SPI Reset (Line 756)
```verilog
.reset(~porb),
```

**Purpose:** Provide active-high reset to SPI module (converts active-low porb to active-high)

**Dependency:** ✅ **Required** - Controls SPI state machine initialization

**Replacement:** Can use inverted external reset

#### Wishbone State Machine Reset (Lines 921-922)
```verilog
always @(posedge wb_clk_i or negedge porb) begin
    if (porb == 1'b0) begin
        xfer_state <= `GPIO_IDLE;
        xfer_count <= 4'd0;
        // ... reset other registers
    end
end
```

**Purpose:** Asynchronous reset of Wishbone bus interface state machine

**Logic:** Triggered on negative edge of porb (transition from 1→0)

**Dependency:** ✅ **Required** - Ensures proper state machine initialization

**Replacement:** Can use external resetb signal for async reset

#### Serial Configuration Reset (Lines 1032-1033)
```verilog
always @(posedge csclk or negedge porb) begin
    if (porb == 1'b0) begin
        // Reset PLL trim settings and serial state
    end
end
```

**Purpose:** Asynchronous reset of serial configuration state machine and PLL trim values

**Logic:** Triggered on negative edge of porb

**Dependency:** ✅ **Required** - Resets PLL configuration to default (slowest) speed

**Note:** This is critical because PLL needs predictable initial state before clock generation

**Replacement:** Can use external resetb signal for async reset

---

### 4. caravel_clocking.v (Clock Distribution & Control)

**Lines with POR:** 24, 51

#### Input Port (Line 24)
```verilog
input porb,    // Master (negative sense) reset from power-on-reset
```

**Purpose:** Receive porb_l from caravel_core

**Dependency:** ✅ **Critical** - Source of reset for clock control logic

#### Reset Combination Logic (Line 51) - **CRITICAL**
```verilog
assign resetb_async = porb & resetb & (!ext_reset);
```

**Purpose:** **Combine three reset sources via AND gate**

**Logic Breakdown:**
- `porb` = POR-generated reset (active low, so 1 = normal operation)
- `resetb` = External reset pad input (active low, so 1 = normal operation)  
- `ext_reset` = Software-triggered reset (active high, so inverted)

**Reset is asserted when ANY of these is true:**
1. `porb = 0` (POR not released) 
2. `resetb = 0` (external reset asserted)
3. `ext_reset = 1` (software reset triggered)

**Dependency:** ✅ **CRITICAL** - Controls master clock output

**Effect when porb=0:**
- Clock output goes low (or stops)
- No clock to CPU/logic
- System in reset state

**Replacement:** Remove `porb &` term → `assign resetb_async = resetb & (!ext_reset);`

---

### 5. chip_io.v (I/O Pad Interface)

**Lines with POR:** 63, 115, 1121, 1198

#### Input Port (Line 63)
```verilog
input  porb_h,
```

**Purpose:** Receive porb_h from caravel_core

**Dependency:** ⚠️ **Partial** - Used for dead code (mprj_io_enh) and SKY130 pad enables

#### Pad Enable Assignment (Line 115)
```verilog
assign mprj_io_enh = {`MPRJ_IO_PADS{porb_h}};
```

**Purpose:** Broadcast porb_h to ALL multi-project I/O pad enable signals

**Effect:** All mprj_io pad enables = porb_h (tied high or low together)

**Status:** ⚠️ **DEAD CODE FOR SCL-180**
- SCL-180 pads (`pc3b03ed_wrapper`) don't have ENABLE pins
- Signal is created but never connected to actual pads
- Legacy code from SKY130 port

**Replacement:** This signal can be removed entirely for SCL-180

#### Reset Pad Enable (Line 1121) - **COMMENTED OUT**
```verilog
.ENABLE_H(porb_h),    // Power-on-reset
```

**Status:** ❌ **Not used** - SKY130 reset pad is commented out

**Current Reset Pad (Line 1129):**
```verilog
pc3d21 resetb_pad (
    .PAD(resetb),
    .CIN(resetb_core_h)
);
```

**Note:** SCL-180 `pc3d21` pad is simple input buffer - NO enable pin

#### Pass-through to mprj_io (Line 1198)
```verilog
.porb_h(porb_h),
```

**Purpose:** Route porb_h to mprj_io module

**Dependency:** ⚠️ **Necessary for current design, but unnecessary for SCL-180**

---

### 6. housekeeping_spi.v (SPI Controller Module)

**Lines with POR:** 756 in housekeeping.v (calls this)

**Note:** SPI module receives active-high reset (`~porb`), can be replaced with external reset

**Usage Pattern:**
```verilog
housekeeping_spi hkspi (
    .reset(~porb),
    // ... other signals
);
```

---

### 7. mgmt_core.v (Management Core - Pass-Through Only)

**Lines with POR:** 83-86, 1828-1829

#### Port Declarations (Lines 83-86)
```verilog
input wire por_l_in,
output wire por_l_out,
input wire porb_h_in,
output wire porb_h_out
```

**Purpose:** Interface for POR signals pass-through

#### Pass-Through Logic (Lines 1828-1829)
```verilog
assign por_l_out = por_l_in;
assign porb_h_out = porb_h_in;
```

**Status:** ✅ **NO ACTUAL LOGIC** - Just wiring, can be completely removed

**Analysis:** This module has no internal logic that depends on POR - it's just routing

---

### 8. mprj_io.v (Multi-Project I/O Module)

**Lines with POR:** 38, 40, 70, 74

#### Input Port (Line 38)
```verilog
input porb_h,
```

**Purpose:** Receive porb_h from chip_io

#### Enable Input (Line 40)
```verilog
input [TOTAL_PADS-1:0] enh,
```

**Purpose:** Pad enable signals (driven by `porb_h` in chip_io line 115)

#### SCL-180 Pad Instantiation (Lines 70, 74)
```verilog
pc3b03ed_wrapper area1_io_pad [AREA1PADS - 1:0] (
    .IN(io_in[AREA1PADS - 1:0]),
    .OUT(io_out[AREA1PADS - 1:0]),
    .PAD(io[AREA1PADS - 1:0]),
    .INPUT_DIS(inp_dis[AREA1PADS - 1:0]),
    .OUT_EN_N(oeb[AREA1PADS - 1:0]),
    .dm(dm[AREA1PADS*3 - 1:0])
);
```

**Critical Finding:** ⚠️ **`enh` signal is NOT connected to pads!**

**Analysis:** 
- Module receives `enh` input (driven by `porb_h`)
- SCL-180 pads don't have ENABLE pins
- `enh` signal is unused
- This is dead code from SKY130 port

---

## Reset Distribution Architecture

### Current POR-Based Distribution

```
dummy_por (caravel_core.v line 1382)
    │
    ├── porb_h (3.3V domain)
    │   ├── chip_io.v line 115 → mprj_io_enh (DEAD CODE - SCL180 pads no enable)
    │   ├── chip_io.v line 1121 → reset pad ENABLE_H (SKY130 - COMMENTED OUT)
    │   ├── mprj_io.v line 38 → module input
    │   └── mgmt_core.v → porb_h_out (PASS-THROUGH)
    │
    ├── porb_l = porb_h (1.8V domain - direct assignment)
    │   ├── caravel_core.v line 528 → housekeeping.v line 82
    │   │   ├── housekeeping.v line 265 → flash_csb_oeb (ACTIVE)
    │   │   ├── housekeeping.v line 267 → flash_clk_oeb (ACTIVE)
    │   │   ├── housekeeping.v line 756 → SPI .reset (ACTIVE)
    │   │   ├── housekeeping.v line 921 → WB state machine (ACTIVE)
    │   │   └── housekeeping.v line 1032 → Serial config reset (ACTIVE)
    │   │
    │   └── caravel_core.v line 591 → caravel_clocking.v line 24
    │       └── caravel_clocking.v line 51 → clock control AND gate (ACTIVE)
    │
    └── por_l = ~porb_l (1.8V domain - inverted)
        ├── vsdcaravel.v line 255 → caravel_core .por input
        └── vsdcaravel.v line 313 → mgmt_core_wrapper .por_l input (PASS-THROUGH)
```

### Active Dependencies (Real POR Usage)

**Only 2 modules have REAL POR dependencies:**

1. **housekeeping.v** (5 usage locations)
   - Flash SPI output control during reset
   - Housekeeping SPI reset
   - Wishbone state machine async reset
   - Serial configuration state machine async reset

2. **caravel_clocking.v** (1 critical usage location)
   - Clock output disable via AND gate with resetb

### Dead Code & Pass-Through

1. **chip_io.v line 115** - `mprj_io_enh` (DEAD - SCL180 pads have no ENABLE pins)
2. **chip_io.v line 1121** - SKY130 reset pad (COMMENTED OUT)
3. **mgmt_core.v** - All POR routing (PASS-THROUGH ONLY)
4. **mprj_io.v** - `enh` input (UNUSED - not connected to pads)

---

## Dependency Mapping

### Which Blocks ACTUALLY Depend on POR?

| Block | Module | Dependencies | Type | Replaceable? |
|-------|--------|--------------|------|-------------|
| **Housekeeping SPI Control** | housekeeping.v | porb_l | Reset signal | ✅ Yes (use resetb) |
| **Housekeeping Flash Control** | housekeeping.v | porb_l | Output disable | ✅ Yes (use resetb) |
| **WB State Machine** | housekeeping.v | porb_l | Async reset | ✅ Yes (use resetb) |
| **Serial Config** | housekeeping.v | porb_l | Async reset | ✅ Yes (use resetb) |
| **Clock Control** | caravel_clocking.v | porb_l | AND gate input | ✅ Yes (remove &) |
| **SPI Reset** | housekeeping_spi | ~porb_l | Active-high reset | ✅ Yes (use ~resetb) |
| **Pad Enables** | chip_io/mprj_io | porb_h | ENABLE_H | ✅ Yes (remove - dead code) |
| **Management Core** | mgmt_core | porb_h/por_l | Pass-through | ✅ Yes (remove) |

**Conclusion:** ALL POR dependencies can be replaced with external `resetb` signal

---

## Critical Findings

### Finding #1: SCL-180 Pads Don't Have ENABLE Pins

**Evidence:**
- `pc3b03ed_wrapper`: `OUT, PAD, IN, INPUT_DIS, OUT_EN_N, dm` (NO ENABLE_H)
- `pc3d01_wrapper`: `IN, PAD` (NO ENABLE)
- `pt3b02_wrapper`: `IN, PAD, OE_N` (NO ENABLE)

**Impact:** `mprj_io_enh` signal driven by `porb_h` is DEAD CODE

---

### Finding #2: Code Explicitly States Level Shifters Are Built-In

**From dummy_por.v line 81:**
```verilog
// since SCL180 has level-shifters already available in I/O pads
assign porb_l = porb_h;
```

**Implication:** No separate level-shifting logic needed - pads handle it internally

---

### Finding #3: External Reset Already Exists and Works

**Evidence from chip_io.v lines 1129-1131:**
```verilog
pc3d21 resetb_pad (
    .PAD(resetb),
    .CIN(resetb_core_h)
);
```

**Evidence from testbench:**
- External reset is driven from testbench
- Chip operates correctly without internal POR
- No POR signals needed for normal operation

---

### Finding #4: Only Two Modules Have Real Dependencies

**Active POR Users:**
1. `housekeeping.v` - 5 usage locations
2. `caravel_clocking.v` - 1 critical location

**All others:** Pass-through or dead code

---

### Finding #5: Reset Pad is Simple Input, Not Complex XRES

**SKY130 Approach (COMMENTED OUT):**
```verilog
sky130_fd_io__top_xres4v2 resetb_pad (
    .ENABLE_H(porb_h),    // Requires POR!
    // ...
);
```

**SCL-180 Approach (ACTIVE):**
```verilog
pc3d21 resetb_pad (
    .PAD(resetb),
    .CIN(resetb_core_h)
);
```

**Implication:** No POR enable needed for reset pad itself

---

## Removal Strategy

### Phase 1: Analysis ✅ COMPLETE

### Phase 2: RTL Modifications (Next Steps)

**Files to Modify:**

1. **caravel_core.v**
   - ❌ Remove `dummy_por` instantiation (lines 1382-1391)
   - ❌ Remove `porb_h`, `porb_l`, `por_l` output declarations (lines 60-61)
   - ✅ Modify `housekeeping` connection: replace `.porb(porb_l)` with `.porb(resetb_n)`
   - ✅ Modify `caravel_clocking` connection: replace `.porb(porb_l)` with `.porb(resetb_n)`

2. **housekeeping.v**
   - ✅ Rename `input porb` to `input resetb` (line 82)
   - ✅ Update all usages of `porb` to `resetb` (lines 265, 267, 756, 921, 1032)

3. **caravel_clocking.v**
   - ✅ Rename `input porb` to `input resetb` (line 24)
   - ✅ Modify line 51: `assign resetb_async = resetb & (!ext_reset);` (remove `porb &`)

4. **chip_io.v**
   - ❌ Remove `input porb_h` (line 63)
   - ❌ Remove `assign mprj_io_enh = {`MPRJ_IO_PADS{porb_h}};` (line 115)
   - ✅ Remove or comment out `.porb_h(porb_h)` in mprj_io connection (line 1198)

5. **vsdcaravel.v**
   - ❌ Remove signal declarations: `porb_h`, `porb_l`, `por_l` (lines 175-177)
   - ✅ Modify caravel_core connections (lines 254-255)
   - ✅ Modify mgmt_core_wrapper connections (lines 312-313)

6. **mprj_io.v**
   - ❌ Remove `input porb_h` (line 38) - if not used elsewhere
   - ✅ Update port connections

7. **mgmt_core.v**
   - ❌ Remove POR port declarations (lines 83-86)
   - ❌ Remove POR pass-through logic (lines 1828-1829)

8. **caravel_netlists.v**
   - ❌ Remove `include "dummy_por.v"` (line 89)

9. **synthesis/output/vsdcaravel_synthesis.v**
   - ❌ Remove `include "dummy_por.v"` (line 8)

---

## Summary Table

| File | Dependency Type | Lines | Action | Impact |
|------|-----------------|-------|--------|--------|
| caravel_core.v | **Generates POR** | 1382 | Remove instantiation | HIGH |
| housekeeping.v | **Active POR User** | 265, 267, 756, 921, 1032 | Replace with resetb | HIGH |
| caravel_clocking.v | **Active POR User** | 51 | Remove from AND gate | HIGH |
| chip_io.v | Dead code | 115, 1121 | Remove | LOW |
| vsdcaravel.v | Distribution | 175-177, 254-255, 312-313 | Update routing | MEDIUM |
| mprj_io.v | Unused | 40 | Remove enh signal handling | LOW |
| mgmt_core.v | Pass-through | 83-86, 1828-1829 | Remove ports | LOW |
| caravel_netlists.v | Include | 89 | Remove | LOW |

---

**Document Version:** 1.0  
**Status:** ✅ ANALYSIS COMPLETE  
**Next Step:** Phase 2 - RTL Modifications  
**Last Updated:** December 15, 2025, 11:36 AM IST

---

## Appendix: Usage Line-by-Line Reference

### Complete Usage Locations

```
FILE: vsdcaravel.v
  L175-177: Signal declarations (porb_h, porb_l, por_l)
  L254-255: caravel_core connections
  L312-313: mgmt_core_wrapper connections

FILE: caravel_core.v
  L60-61:   Output declarations
  L528:     housekeeping connection
  L591:     caravel_clocking connection
  L1382:    dummy_por instantiation [MAIN SOURCE]

FILE: housekeeping.v
  L82:      porb input declaration
  L265:     flash_csb_oeb control
  L267:     flash_clk_oeb control
  L756:     SPI module reset
  L921-922: WB state machine reset
  L1032:    Serial config reset

FILE: caravel_clocking.v
  L24:      porb input declaration
  L51:      resetb_async logic [CRITICAL AND GATE]

FILE: chip_io.v
  L63:      porb_h input
  L115:     mprj_io_enh assignment [DEAD CODE]
  L1121:    Reset pad ENABLE_H [COMMENTED OUT]
  L1198:    mprj_io connection

FILE: mprj_io.v
  L38:      porb_h input
  L40:      enh input
  L70, 74:  SCL180 pad instantiation

FILE: mgmt_core.v
  L83-86:   POR port declarations
  L1828:    por_l_out assignment
  L1829:    porb_h_out assignment

FILE: caravel_netlists.v
  L89:      `include "dummy_por.v"

FILE: synthesis/output/vsdcaravel_synthesis.v
  L8:       `include "dummy_por.v"
  L60486:   dummy_por instantiation
```

