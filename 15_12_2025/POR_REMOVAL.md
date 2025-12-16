# 🎯 **POR REMOVAL IMPLEMENTATION PLAN**

***

## 🔍 **Key Discovery**

**Introduce a single top-level reset pins:** `resetn`

***

## 📝 **COMPLETE MODIFICATION PLAN**

## Step 1: Made the Dummy_por module commented 

<img width="714" height="596" alt="image" src="https://github.com/user-attachments/assets/1656ca3e-b592-4d47-bf9a-578c263cd913" />


### Step 2: In `rtl/vsdcaravel.v` (Top Module)

- Declare `resetn` as input

<img width="714" height="219" alt="image" src="https://github.com/user-attachments/assets/283476b8-79c4-477d-b03b-b65e33d5fa80" />

- Assign por signals with external signal `resetn`

```verilog
assign porb_h = resetn;
assign porb_l = resetn;
assign por_l = ~resetn;
```
<img width="717" height="255" alt="image" src="https://github.com/user-attachments/assets/09663251-8685-4546-bda0-77329c4a9c51" />

### Step 3: In `rtl\caravel_core,v`

- Declare `porb_l` as output port

<img width="385" height="252" alt="image" src="https://github.com/user-attachments/assets/04df801c-621e-427b-b656-d5e345a76e24" />

- Comment out or remove the dummy_por

<img width="520" height="263" alt="image" src="https://github.com/user-attachments/assets/471ece8f-9ef9-431d-adcd-d1d5df092dca" />

### Step 4: In `rtl\caravel_netlist.v`

- Comment out ``include "dummy_por.v"`

<img width="718" height="596" alt="image" src="https://github.com/user-attachments/assets/105b287e-cfd4-42d1-9014-6ffc41c8d872" />


### Step 5: In `dv/hkspi/hkspi_tb.v`

- Declare `resetn` as register
  
<img width="716" height="198" alt="image" src="https://github.com/user-attachments/assets/7bcf61b4-9f78-4cd6-a932-5ba883e080f1" />
  
- Make the `resetn` intialize to `0`
- Increase the delay to `#20000` and `#10000`
- Then After `#20000` delay make the `resetn` as `1`

<img width="716" height="371" alt="image" src="https://github.com/user-attachments/assets/bca18d09-81c6-499f-9c0e-4a692a2a9aa2" />

- Instantiate the `resetn` in `vsdcaravel`

<img width="700" height="642" alt="image" src="https://github.com/user-attachments/assets/f8e63abb-8fdb-4588-87ba-4f8b3664633c" />


#### Note: Change the `hkspi_tb.v` in gls as above

### Step 5: Test
```bash
cd dv/hkspi
make clean && make
./simv
gtkwave hkspi.vcd
```

<img width="1580" height="989" alt="image" src="https://github.com/user-attachments/assets/0e8d7a84-1705-47a1-ab81-3f3bb578162c" />


## ✅ Result
```
Monitor: Test HK SPI (RTL) Passed
```
<img width="736" height="645" alt="image" src="https://github.com/user-attachments/assets/c464f93c-d4da-433b-a6d5-8cad7378cf3b" />


All 19 SPI register reads succeed! 🎉


# Removed on-chip POR; migrated to external reset-only architecture
