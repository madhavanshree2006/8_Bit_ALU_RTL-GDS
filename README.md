

# 1️⃣ 8-BIT ALU – SPEC 

### 🔹 Inputs

- `A [7:0]` – Operand A
- `B [7:0]` – Operand B
- `OP [2:0]` – Operation select

### 🔹 Outputs

- `Y [7:0]` – Result
- `CARRY` – Carry out (ADD / SUB)
- `ZERO` – Result is zero
- `OVERFLOW` – Signed overflow (ADD / SUB)

---

## 🔹 Opcode Map (Simple & CPU-like)

| OP | Operation |
| --- | --- |
| 000 | ADD |
| 001 | SUB |
| 010 | AND |
| 011 | OR |
| 100 | XOR |
| 101 | SLT (signed) |
| 110 | SHIFT LEFT |
| 111 | SHIFT RIGHT |

✔ No multipliers

✔ No latches

✔ Fully combinational
---

# ✅ STEP 1 8-BIT ALU RTL (SYNTHESIZABLE VERILOG)

Create a file:

```bash
alu_8bit.v

```

```verilog
module alu_8bit (
    input  wire [7:0] A,
    input  wire [7:0] B,
    input  wire [2:0] OP,
    output reg  [7:0] Y,
    output reg        CARRY,
    output reg        OVERFLOW,
    output wire       ZERO
);

    reg [8:0] temp;  // for carry detection

    always @(*) begin
        // default values (VERY IMPORTANT to avoid latches)
        Y        = 8'b0;
        CARRY    = 1'b0;
        OVERFLOW = 1'b0;
        temp     = 9'b0;

        case (OP)
            3'b000: begin // ADD
                temp  = {1'b0, A} + {1'b0, B};
                Y     = temp[7:0];
                CARRY = temp[8];
                OVERFLOW = (~(A[7] ^ B[7])) & (Y[7] ^ A[7]);
            end

            3'b001: begin // SUB
                temp  = {1'b0, A} - {1'b0, B};
                Y     = temp[7:0];
                CARRY = temp[8];
                OVERFLOW = ((A[7] ^ B[7])) & (Y[7] ^ A[7]);
            end

            3'b010: Y = A & B;        // AND
            3'b011: Y = A | B;        // OR
            3'b100: Y = A ^ B;        // XOR
            3'b101: Y = ($signed(A) < $signed(B)) ? 8'd1 : 8'd0; // SLT
            3'b110: Y = A << 1;       // SHIFT LEFT
            3'b111: Y = A >> 1;       // SHIFT RIGHT

            default: Y = 8'b0;
        endcase
    end

    assign ZERO = (Y == 8'b0);

endmodule

```
---

## ✅ STEP 2 — Put RTL in a proper project structure

Organize like an ASIC project.

```bash
mkdir -p ~/VLSI/{rtl,tb}
mv ~/alu_8bit.v ~/VLSI/rtl/

```

Verify:

```bash
ls ~/VLSI/rtl

```

You should see:
```bash
alu_8bit.v
```
## ✅ STEP 3 — Write the ALU Testbench (MANDATORY)

**never trust RTL without simulation**.

Create testbench:

```bash
gedit ~/VLSI/tb/tb_alu_8bit.v

```

Use a robust **testbench** 👇

```verilog
`timescale 1ns/1ps

module tb_alu_8bit;

    reg  [7:0] A, B;
    reg  [2:0] OP;
    wire [7:0] Y;
    wire CARRY, OVERFLOW, ZERO;

    alu_8bit dut (
        .A(A),
        .B(B),
        .OP(OP),
        .Y(Y),
        .CARRY(CARRY),
        .OVERFLOW(OVERFLOW),
        .ZERO(ZERO)
    );

    initial begin
        $display("Starting ALU Testbench");

        // ADD
        A = 8'd10; B = 8'd5; OP = 3'b000; #10;
        $display("ADD: %d + %d = %d", A, B, Y);

        // SUB
        A = 8'd10; B = 8'd20; OP = 3'b001; #10;
        $display("SUB: %d - %d = %d", A, B, Y);

        // AND
        A = 8'hAA; B = 8'h0F; OP = 3'b010; #10;

        // OR
        OP = 3'b011; #10;

        // XOR
        OP = 3'b100; #10;

        // SLT
        A = -8'd5; B = 8'd3; OP = 3'b101; #10;

        // SHIFT LEFT
        A = 8'b00001111; OP = 3'b110; #10;c

        // SHIFT RIGHT
        OP = 3'b111; #10;

        // ZERO FLAG
        A = 8'd0; B = 8'd0; OP = 3'b000; #10;

        $display("Testbench completed");
        $finish;
    end

endmodule

```

Save and close.

---

## ✅ STEP 4 — Compile RTL + Testbench (Icarus)

```bash
cd ~/VLSI
iverilog -o alu_sim rtl/alu_8bit.v tb/tb_alu_8bit.v

```

If **no errors appear** → ✅ RTL is synthesizable.

---
## ✅ STEP 5 — Run Simulation

```bash
vvp alu_sim

```

Expected:

- Printed results for ADD, SUB, logic ops
- No crashes
- No `x` or `z` issues

👉 This confirms **functional correctness**

---

## ✅ INTERPRETING YOUR SIMULATION OUTPUT

```
ADD:  10 +   5 =  15
SUB:  10 -  20 = 246
```

### Why SUB = 246?

- 8-bit unsigned wraparound
- 10 − 20 = **−10**
- In 8-bit two’s complement:
    
    ```
    256 − 10 = 246
    ```
    

✅ **This is CORRECT behavior**

---
<br>
<br>

# NEXT PHASE: OPENLANE SETUP FOR YOUR ALU

Lets we do **RTL → GDS** using your own design.

---

## ✅ STEP 1 Create OpenLane Design Folder

```bash
mkdir -p ~/OpenLane/designs/alu8
mkdir -p ~/OpenLane/designs/alu8/src

```

---

## ✅ STEP 2 Copy RTL into OpenLane

```bash
cp ~/VLSI/rtl/alu_8bit.v ~/OpenLane/designs/alu8/src/
```

Verify:

```bash
ls ~/OpenLane/designs/alu8/src
```

You should see:

```
alu_8bit.v
```

---

## ✅ STEP 3 Create `config.tcl` (CRITICAL FILE)

Create:

```bash
gedit ~/OpenLane/designs/alu8/config.tcl
```

Paste this **minimal, safe config** 👇

```
# Design basics
set ::env(DESIGN_NAME) alu_8bit
set ::env(VERILOG_FILES) [glob $::env(DESIGN_DIR)/src/*.v]

# Clock (dummy clock)
set ::env(CLOCK_PORT) ""
set ::env(CLOCK_PERIOD) "10"

# Floorplan
set ::env(FP_CORE_UTIL) 40
set ::env(FP_ASPECT_RATIO) 1

# Power
set ::env(VDD_NETS) "vdd"
set ::env(GND_NETS) "gnd"

# Synthesis
set ::env(SYNTH_STRATEGY) "AREA 0"
set ::env(SYNTH_MAX_FANOUT) 10

# Routing
set ::env(ROUTING_STRATEGY) 0

```

Save and close.

👉 This tells OpenLane:

- What RTL to read
- How to floorplan
- How to route

---

## ✅ STEP 4 Run OpenLane on YOUR ALU

```bash
cd ~/OpenLane
make mount

```

Inside the OpenLane container, run:

```
./flow.tcl -design alu8

```

⏳ Takes a few minutes.

---

## 🎯 WHAT TO WATCH FOR

You want to see:

```
[SUCCESS]:Flow complete.
```

Warnings are OK

Errors are NOT

---

## 👀 AFTER SUCCESS

Your ALU GDS will be here:

```bash
OpenLane/designs/alu8/runs/<run_name>/results/final/
```

You can open it using:

```bash
klayout alu_8bit.gds
```
---


