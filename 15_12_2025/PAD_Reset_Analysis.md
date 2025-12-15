# SCL-180 PAD Reset Analysis

**Date:** December 15, 2025  
**Author:** Prakhan  
**Design:** VSD RISC-V SoC on SCL-180 PDK  
**Goal:** Justify that an **external reset-only** architecture is safe, and that **on‑chip POR is not required** for SCL‑180 pads.

***

## 1. Reset Pad Behavior in SCL‑180

### 1.1 Actual Reset Pad Used (pc3d21)

In `rtl/chip_io.v` the active reset pad is:

```verilog
pc3d21 resetb_pad (
       .PAD(resetb),
       .CIN(resetb_core_h)
       );
```

This pad has only a pad pin (`PAD`) and a core input (`CIN`), with **no enable, POR, or power gating pins** connected. 

Immediately above, the SKY130 XRES pad usage (now commented out) shows a very different interface:

```verilog
/* sky130_fd_io__top_xres4v2 resetb_pad (
    ...
    .ENABLE_H(porb_h),       // Power-on-reset
    .EN_VDDIO_SIG_H(...),
    .INP_SEL_H(...),
    .FILT_IN_H(...),
    .PULLUP_H(...),
    ...
); */
```

This clearly shows that **only the SKY130 reset pad needed a POR‑driven enable**, while the SCL‑180 `pc3d21` pad is a simple input buffer without any POR/control pins. 

**Answer to questions for SCL‑180 reset pad:**

- **Internal enable required?**  
  No. `pc3d21` exposes only `PAD` and `CIN` in this design; there is no `.ENABLE_H` or similar control. 

- **POR‑driven gating required?**  
  No. There is no POR‑related port on the instantiated pad, and no POR signal is connected into it. 

- **Is the reset pin asynchronous?**  
  Functionally yes: the reset net (`resetb_core_h`) goes directly into asynchronous reset logic (e.g., async resets in housekeeping and clocking), so it is treated as an async external reset.  

- **Available immediately after VDD?**  
  Since `pc3d21` is instantiated only as a buffer with no enable or power‑good inputs, the design assumes the pad output is usable as soon as VDD and the external reset source are valid. There is no additional RTL gating that delays its usability. 

- **Are there power‑up sequencing constraints that mandate a POR?**  
  Within this repo there are **no pad‑level comments, no PDK notes, and no additional logic** that impose a POR requirement for `pc3d21`; the only sequencing constraint is that external reset (`resetb`) must be held asserted while supplies ramp and then released cleanly.  

***

## 2. General SCL‑180 I/O Pad Interfaces vs SKY130

### 2.1 SCL‑180 I/O Pads (pc3b03ed / pc3d01 / pt3b02)

From `rtl/scl180_wrapper/*.v`: 

- **pc3b03ed_wrapper** (bidirectional pad):

  ```verilog
  module pc3b03ed_wrapper(OUT, PAD, IN, INPUT_DIS, OUT_EN_N, dm);
  output IN;
  input  OUT, INPUT_DIS, OUT_EN_N;
  inout  PAD;
  input  [2:0] dm;
  ...
  pc3b03ed pad(.CIN(IN),
               .OEN(output_EN_N),
               .RENB(pull_down_enb),
               .I(OUT),
               .PAD(PAD));
  endmodule
  ```

- **pc3d01_wrapper** (input pad):

  ```verilog
  module pc3d01_wrapper(output IN, input PAD);
      pc3d01 pad(.CIN(IN), .PAD(PAD));
  endmodule
  ```

- **pt3b02_wrapper** (tristate output):

  ```verilog
  module pt3b02_wrapper(output IN, inout PAD, input OE_N);
      pt3b02 pad(.CIN(IN), .OEN(OE_N), .I(), .PAD(PAD));
  endmodule
  ```

**Key observations for SCL‑180 pads:**

- No `ENABLE_H`, `ENABLE_VDDA_H`, or `ENABLE_VSWITCH_H` pins in any wrapper. 
- Control pins are limited to `INPUT_DIS`, `OUT_EN_N`/`OE_N`, and drive‑mode `dm` bits. 
- There is **no POR‑related pin to connect `porb_h`/`porb_l` into** the SCL‑180 pad macros. 

In contrast, `rtl/pads.v` still contains legacy SKY130 macros with POR‑controlled enables, e.g.: 

```verilog
.ENABLE_H(porb_h),
.ENABLE_INP_H(...),
.ENABLE_VDDA_H(porb_h),
.ENABLE_VSWITCH_H(...),
.ENABLE_VDDIO(...),
```

but those SKY130 pads are not used in this SCL‑180 build; all the actual instantiations for GPIO and flash use the SCL‑180 `pc3b03ed_wrapper` / `pt3b02_wrapper` interfaces with **no ENABLE_H pins**. 

### 2.2 SCL‑180 Pads and Level Shifting

`rtl/dummy_por.v` explicitly states: 

> since SCL180 has level‑shifters already available in I/O pads  
> `assign porb_l = porb_h;`

This shows that **voltage‑domain handling is inside the pad cells**; the POR logic does **not** perform level shifting. The POR signals are just used as digital resets in the core. 

***

## 3. SKY130: Why POR Was Mandatory There

In the original Caravel (SKY130):

- GPIO pads and XRES reset pad have multiple ENABLE pins that must be driven correctly after power‑up.
- Example, from commented block in `rtl/chip_io.v`: 

  ```verilog
  sky130_fd_io__top_xres4v2 resetb_pad (
      ...
      .ENABLE_H(porb_h),      // Power-on-reset
      .EN_VDDIO_SIG_H(...),
      .INP_SEL_H(...),
      .FILT_IN_H(...),
      .PULLUP_H(...),
      ...
  );
  ```

- General GPIO macros (see `rtl/pads.v`) use:

  ```verilog
  .ENABLE_H(porb_h),
  .ENABLE_VDDA_H(porb_h),
  .ENABLE_VSWITCH_H(...),
  .ENABLE_VDDIO(...),
  ```

**Implications in SKY130:**

- Pads are **not fully “on” at power‑up**; they require enable sequencing via `ENABLE_H`/`ENABLE_VDDA_H`.
- Those enables are **tied to POR** (`porb_h`), so the analog POR macro must guarantee a clean, timed transition to enable the pads only after supplies are in range. 
- The XRES pad itself depends on POR for its internal glitch‑filter and pullups to behave correctly. 

Therefore, in SKY130:

- A **dedicated POR macro is architecturally necessary**, because pad behavior and reset pad behavior explicitly depend on POR‑driven enables.
- Using only an external reset pin without proper POR‑driven pad enables would violate the intended power‑up sequence.

***

## 4. Why POR is Not Mandatory in SCL‑180

Given the evidence above, the situation on SCL‑180 is very different:

1. **SCL‑180 pads have no ENABLE_H‑style pins**

   - All active SCL‑180 pad wrappers (`pc3b03ed_wrapper`, `pc3d01_wrapper`, `pt3b02_wrapper`, `pc3d21`) expose only basic data/enable pins (OE_N/OUT_EN_N, INPUT_DIS), not POR‑controlled ENABLE pins. 
   - Legacy SKY130 macros with `.ENABLE_H(porb_h)` still exist in `rtl/pads.v` but are not instantiated in the SCL‑180 build. 

2. **Reset pad `pc3d21` is a simple input buffer**

   - The instantiated reset pad only has `.PAD(resetb)` and `.CIN(resetb_core_h)`; there is no POR enable or power‑good pin. 
   - A comment in `chip_io.v` notes that the old analog reset pad (XRES) and on‑chip POR were replaced with a direct digital reset input due to lack of on‑board POR, and that the XRES pad had been previously used for a glitch‑free reset in SKY130. 

3. **Level shifting handled inside pads**

   - `dummy_por.v` explicitly notes SCL‑180 pads already contain level shifters and directly wires `porb_l = porb_h` for simulation. 
   - This means POR is not part of level‑shift sequencing; it is just a digital reset convenience in this RTL.

4. **No pad‑level power‑up constraints referencing POR**

   - The repository has no PDK PDF or pad datasheet bundled, but there are **no comments or RTL glue** enforcing a particular power‑up sequence tied to POR on SCL‑180.  
   - All POR usage is in digital logic (housekeeping, clocking) and not in pad instantiations in the SCL‑180 configuration.  

5. **Design already assumes external reset is responsible for sequencing**

   - Comments in `chip_io.v` state that the digital reset input is used “due to the lack of an on‑board power‑on‑reset circuit,” explicitly shifting responsibility to the external reset pad and external circuit. 
   - The hkspi testbench drives `RSTB` directly from the testbench to control reset; there is no reliance on internal POR for the design to start correctly. 

**Conclusion:**

- In SCL‑180, **pads come up in a usable state as soon as power is valid**, without any explicit POR enable pins tied into the design. 
- The only requirement is that external reset is held active until VDD is within spec and clocks are stable, which is standard practice handled by an external supervisor or RC reset circuit.
- Therefore, **an external reset‑only strategy is both safe and architecturally sufficient on SCL‑180**, whereas SKY130 required POR due to pad‑interface constraints.

***

## 5. Direct Answers to Task Questions

1. **Does the reset pad require an internal enable?**  
   No. The instantiated SCL‑180 reset pad (`pc3d21`) has only `PAD` and `CIN` ports; there is no enable or POR port wired into it. 

2. **Does the reset pad require POR‑driven gating?**  
   No. The original SKY130 XRES pad required `.ENABLE_H(porb_h)`, but that instance is commented out, and the SCL‑180 pad replacement does not expose POR‑related controls. 

3. **Is the reset pin asynchronous?**  
   Yes in practice. The core reset (`resetb_core_h`) is used as an asynchronous reset in downstream logic (e.g., async reset in housekeeping and clock control), so the design treats it as an async external reset.  

4. **Is the reset pin available immediately after VDD?**  
   From the RTL view, yes: there is no extra enable or POR gating; once VDD and the external reset source are valid, the pad simply buffers `resetb` to `resetb_core_h`. Any additional analog behavior (e.g., ESD, internal filtering) is inside the SCL‑180 pad macro and not exposed here. 

5. **Are there documented power‑up sequencing constraints that mandate a POR?**  
   None appear in this repository. There are no pad comments, timing annotations, or additional logic that require an on‑chip POR; instead, comments explicitly say the design uses a digital reset input because there is no on‑chip POR.  

6. **Why was POR mandatory in SKY130 but not in SCL‑180?**

   - In SKY130:
     - Pad macros (GPIO and XRES) have `ENABLE_H`/`ENABLE_VDDA_H` pins that must be sequenced based on a POR signal. 
     - The reset pad itself relies on POR enable to provide a glitch‑free, filtered reset. 
     - Therefore, **POR is architecturally required** to bring pads and reset into a known state.

   - In SCL‑180:
     - Pad macros (pc3b03ed, pc3d01, pt3b02, pc3d21) do **not** expose ENABLE_H‑type pins in this integration. 
     - Level shifting is handled inside the pad macros, not by POR logic. 
     - The reset pad is a simple buffer that does not depend on POR enable. 
     - Therefore, **POR is not needed at the pad level**, and an external reset‑only scheme is sufficient, as long as board‑level circuitry asserts reset correctly at power‑up.

***

## 6. References

- `rtl/chip_io.v` – reset pad instantiation and commented SKY130 XRES usage. 
- `rtl/scl180_wrapper/pc3b03ed_wrapper.v`, `pc3d01_wrapper.v`, `pt3b02_wrapper.v` – SCL‑180 pad wrapper interfaces (no ENABLE_H pins). 
- `rtl/dummy_por.v` – comment on SCL‑180 pads having built‑in level shifters. 
- `rtl/pads.v` – legacy SKY130 pad macros with `.ENABLE_H(porb_h)` and related enables. 
- `rtl/caravel_netlists.v` – inclusion of `pc3d21.v` as the reset pad cell. 
- Testbench and documentation comments indicating reliance on external reset (`resetb`).  

***
