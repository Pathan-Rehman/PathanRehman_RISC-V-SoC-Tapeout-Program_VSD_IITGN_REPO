# Removal of On-Chip POR and Final GLS Validation (SCL-180)
# **Task 3**

## **Main Goal:**
Remove the on-chip `dummy_por` module and prove that using only an **external reset pin** is safe for SCL-180. 

***

## **How `dummy_por` Worked (The Problem):**

The original design used a **behavioral POR module**: 

```verilog
module dummy_por(
    output porb_h,  // 3.3V domain reset
    output porb_l,  // 1.8V domain reset  
    output por_l    // Active-high reset
);
```

**Issues with this approach:**
- **Not synthesizable** - all logic inside `ifdef SIM` blocks 
- **Fake delay** - used 500ns simulation delay (real POR needs ~15ms) 
- **Generated 3 reset signals** from power rails using Schmitt triggers 
- **Drove critical blocks**: housekeeping, clock control, SPI modules 
- **Created blackbox** in synthesis that can't be fabricated 

***

## **Your Implementation (The Solution):**

### **Step 1: External Reset Pin**
- Added `resetn` input to top module `vsdcaravel.v` 
- Directly mapped all POR signals to this external pin:
  ```verilog
  assign porb_h = resetn;
  assign porb_l = resetn;
  assign por_l = ~resetn;
  ```

### **Step 2: Removed `dummy_por`**
- Commented out instantiation in `caravel_core.v` 
- Removed from netlist includes 

### **Step 3: Modified Testbench**
- Declared `resetn` as register 
- Controlled reset timing externally:
  ```verilog
  initial resetn = 0;
  #20000 resetn = 1;  // Release reset after delay
  ```

### **Step 4: Connected to Top Module**
- Instantiated `resetn` in `vsdcaravel` module 

***

## **Why This Works for SCL-180:**

**Key Discovery:** SCL-180 pads are fundamentally different from SKY130: 

| **Feature** | **SKY130** | **SCL-180** |
|------------|------------|-------------|
| **Pad enables** | Requires `.ENABLEH(porb_h)` | No ENABLE pins |
| **Reset pad** | Complex XRES with POR gating | Simple `pc3d21` buffer |
| **Level shifters** | External logic needed | Built into pads |
| **POR requirement** | **Mandatory** | **Not needed** |

**Evidence from code:**
- SCL-180 `pc3d21` reset pad has only `.PAD` and `.CIN` - no POR control 
- Comment in `dummy_por.v` line 81: *"SCL180 has level-shifters already available in IO pads"* 
- All GPIO wrappers (`pc3b03ed`, `pc3d01`) have no ENABLEH pins 

***

## **Result:**
✅ **External reset-only architecture is safe** because SCL-180 pads:
1. Don't need POR-driven enable sequencing
2. Have built-in level shifters
3. Are usable immediately after VDD is stable 

✅ **Verification:** RTL and GLS simulations passed, all 19 SPI registers functional 
