# VSDBabySoC – Data Flow Between Modules

VSDBabySoC integrates a **RISC-V processor (RVMYTH)**, a **DAC (AVSDDAC)**, and a **PLL (AVSDPLL)** to convert digital computations into analog output.

---

##  Data Flow

### PLL Module – Clock Generation

```verilog
wire clk;
avsddpll pll_inst(
    .CLK_IN(ext_clk),
    .CLK_OUT(clk)
);
```

- Generates stable clock (clk) for the CPU and DAC.
- Ensures synchronous operation across the SoC.

---

## Pipeline Stages

```bash
| Stage | Module | Operation |
|--------|---------|------------|
| @0–@1 | IMEM | Fetch instruction |
| @1–@2 | CPU Decode | Decode opcode, identify rs1, rs2, rd |
| @2–@3 | RF | Read operands |
| @3 | ALU | Execute instruction |
| @4–@5 | DMEM | Load/Store data |
| @3 | RF | Write result back |
```
---

## Role of x17 (R17) in Driving the DAC

In VSDBabySoC, **x17 (register 17)** serves as the bridge between the CPU computations and the analog output DAC.

## How x17 is Updated

1. **Instruction Selection**  

   Instructions that target x17 have `rd = 5'd17`. 
   Examples:
   ```assembly
   ADDI x17, x0, 0   // Initialize x17
   ADD  x17, x17, x11
   SUB  x17, x17, x11
   ```
- rd field in instruction = 17 → CPU knows the destination register.

2. **ALU Execution**

- Arithmetic/logic operations are performed in the ALU stage.
- Result of operation is stored in a temporary pipeline register.

3.**Writeback**

- Result from ALU (or memory if it’s a load) is written back to the register file at stage @5.

```assembly
rf_wr_en = (rd_valid && valid && rd != 5'b0) || valid_load;
rf_wr_index = rd;         // rd = 5'd17 for x17
rf_wr_data  = result;     // ALU or memory result
```
- Only when rf_wr_en is high does the CPU update x17.

---

## How DAC Reads x17

- The DAC takes the lower 10 bits of x17 as its input.

```verilog
always @(posedge CLK) begin
    OUT = CPU_Xreg_value_a5[17][9:0]; // LSBs of x17
end
```

- CPU_Xreg_value_a5[17] → value of x17 after writeback stage.
- The DAC converts this 10-bit value into analog voltage:

```Formula
OUT = VREFL + (D / 1023.0) * (VREFH - VREFL);
```
---

## Summary of x17 → DAC Flow

- Instruction executes and targets x17 (rd = 17).
- ALU computes the result based on instruction operands.
- Result is written back to x17.
- DAC reads the lower 10 bits of x17 on the next clock.
- DAC outputs the corresponding analog voltage.

**Key point:** x17 acts as the CPU’s “analog output register,” directly controlling the DAC output. Any update to x17 is immediately reflected in the DAC output at the next clock cycle.

This explains the **entire responsibility of x17 in DAC communication** — from instruction selection → ALU computation → writeback → DAC read → analog output.

---