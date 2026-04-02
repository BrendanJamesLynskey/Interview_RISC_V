# Problem 02: Forwarding MUX Logic Design for a 5-Stage Pipeline

## Difficulty
Intermediate to Advanced (covers Tier 2 + Tier 3 material)

## Prerequisites
- 5-stage pipeline registers (IF/ID, ID/EX, EX/MEM, MEM/WB) and their fields
- Forwarding conditions: EX/MEM -> EX and MEM/WB -> EX
- Double-hazard priority
- SystemVerilog syntax for combinational logic

---

## Problem Statement

You are designing the forwarding unit for a 5-stage RISC-V pipeline. The pipeline register fields visible to the forwarding unit are:

```
ID/EX pipeline register (instruction currently entering EX):
  id_ex_rs1       [4:0]   -- source register 1 index
  id_ex_rs2       [4:0]   -- source register 2 index
  id_ex_rs1_data  [31:0]  -- value read from register file for rs1
  id_ex_rs2_data  [31:0]  -- value read from register file for rs2
  id_ex_alu_src   [0:0]   -- 1 if ALU input B is the immediate (not rs2)

EX/MEM pipeline register (instruction currently in MEM):
  ex_mem_reg_write [0:0]  -- 1 if this instruction writes a register
  ex_mem_rd        [4:0]  -- destination register index
  ex_mem_alu_result[31:0] -- ALU result

MEM/WB pipeline register (instruction currently in WB):
  mem_wb_reg_write [0:0]
  mem_wb_rd        [4:0]
  mem_wb_alu_result[31:0] -- ALU result (for non-load instructions)
  mem_wb_read_data [31:0] -- memory read data (for load instructions)
  mem_wb_mem_to_reg[0:0]  -- 1 = use mem read data; 0 = use ALU result
```

**Tasks:**

1. Write the complete combinational logic for `ForwardA` and `ForwardB` (2-bit MUX selects) and the `alu_input_a` and `alu_input_b` MUX outputs in SystemVerilog.
2. Extend the MUX to support a third forwarding case: the MEM/WB result for a **load instruction** (i.e., select `mem_wb_read_data` when `mem_wb_mem_to_reg == 1`). This must be incorporated into the single MUX rather than handled as a special case elsewhere.
3. For the following instruction sequence, manually trace the forwarding MUX select signals for each instruction as it enters the EX stage:

```
Cycle 1-5:  ADD  x1, x2, x3    # writes x1
Cycle 2-6:  SUB  x4, x1, x5    # reads x1 (RAW, 1 cycle apart)
Cycle 3-7:  AND  x6, x1, x4    # reads x1 (RAW, 2 cycles apart); reads x4 (RAW, 1 cycle)
Cycle 4-8:  OR   x7, x6, x8    # reads x6 (RAW, 1 cycle apart)
```

4. The double-hazard case: For the sequence below, explain which value each forwarding source holds in the relevant cycle, and prove that the priority logic selects the correct value.

```
I1: ADD x1, x2, x3    # x1 = 10
I2: ADD x1, x4, x5    # x1 = 20  (overwrites x1)
I3: ADD x6, x1, x7    # must use x1 = 20 (from I2, not x1 = 10 from I1)
```

---

## Solution

### Part 1: ForwardA and ForwardB Logic in SystemVerilog

```systemverilog
module forwarding_unit (
    // ID/EX: instruction entering EX stage (the consumer)
    input  logic [4:0]  id_ex_rs1,
    input  logic [4:0]  id_ex_rs2,

    // EX/MEM: instruction currently in MEM (one stage ahead of consumer)
    input  logic        ex_mem_reg_write,
    input  logic [4:0]  ex_mem_rd,
    input  logic [31:0] ex_mem_alu_result,

    // MEM/WB: instruction currently in WB (two stages ahead of consumer)
    input  logic        mem_wb_reg_write,
    input  logic [4:0]  mem_wb_rd,
    input  logic [31:0] mem_wb_alu_result,
    input  logic [31:0] mem_wb_read_data,
    input  logic        mem_wb_mem_to_reg,

    // Register file outputs (no-hazard case)
    input  logic [31:0] id_ex_rs1_data,
    input  logic [31:0] id_ex_rs2_data,

    // ALU inputs (resolved, forwarded values)
    output logic [31:0] alu_input_a,
    output logic [31:0] alu_input_b,

    // Forwarding select signals (for testability / debug)
    output logic [1:0]  forward_a,
    output logic [1:0]  forward_b
);

    // Forward select encoding:
    //   2'b00 = no forward: use register file output
    //   2'b01 = MEM/WB forward (2-cycle-old result)
    //   2'b10 = EX/MEM forward (1-cycle-old result, highest priority)

    // MEM/WB result selection: ALU result or memory read data (for loads)
    logic [31:0] mem_wb_result;
    assign mem_wb_result = mem_wb_mem_to_reg ? mem_wb_read_data : mem_wb_alu_result;

    // ---- ForwardA: select source for ALU input A ----
    always_comb begin
        forward_a = 2'b00;  // default: no forwarding

        // MEM/WB check first (lower priority; overridden by EX/MEM below)
        if (mem_wb_reg_write &&
            (mem_wb_rd != 5'b0) &&
            (mem_wb_rd == id_ex_rs1)) begin
            forward_a = 2'b01;
        end

        // EX/MEM check (higher priority; override MEM/WB if both match)
        if (ex_mem_reg_write &&
            (ex_mem_rd != 5'b0) &&
            (ex_mem_rd == id_ex_rs1)) begin
            forward_a = 2'b10;
        end
    end

    // ---- ForwardB: select source for ALU input B ----
    always_comb begin
        forward_b = 2'b00;

        if (mem_wb_reg_write &&
            (mem_wb_rd != 5'b0) &&
            (mem_wb_rd == id_ex_rs2)) begin
            forward_b = 2'b01;
        end

        if (ex_mem_reg_write &&
            (ex_mem_rd != 5'b0) &&
            (ex_mem_rd == id_ex_rs2)) begin
            forward_b = 2'b10;
        end
    end

    // ---- ALU input A MUX ----
    always_comb begin
        case (forward_a)
            2'b00:   alu_input_a = id_ex_rs1_data;     // register file
            2'b01:   alu_input_a = mem_wb_result;       // MEM/WB (includes load data)
            2'b10:   alu_input_a = ex_mem_alu_result;   // EX/MEM ALU result
            default: alu_input_a = id_ex_rs1_data;
        endcase
    end

    // ---- ALU input B MUX ----
    // Note: the actual ALU input B also has a second MUX for immediate vs. rs2.
    // This module produces the "forwarded rs2 value" which is then fed into
    // the ALU-src MUX downstream. Some implementations combine both MUXes here.
    always_comb begin
        case (forward_b)
            2'b00:   alu_input_b = id_ex_rs2_data;
            2'b01:   alu_input_b = mem_wb_result;
            2'b10:   alu_input_b = ex_mem_alu_result;
            default: alu_input_b = id_ex_rs2_data;
        endcase
    end

endmodule
```

**Key design decisions explained:**

The `mem_wb_result` signal consolidates the load/ALU selection for MEM/WB forwarding. When the MEM/WB instruction is a load (`mem_wb_mem_to_reg == 1`), the forwarded value must be the memory read data, not the ALU result (which would be the load *address* rather than the loaded value). This consolidation means no special handling is required in the forwarding MUX itself — it simply selects `mem_wb_result` for the 2'b01 case.

The EX/MEM path always forwards the ALU result (`ex_mem_alu_result`). For the load-use hazard case, the load instruction is in MEM when the dependent instruction is stalled in ID (one stall cycle injected). After the stall, the load is in MEM/WB and its data is forwarded via the `mem_wb_result` path. This means EX/MEM forwarding of a *load's memory data* is never needed — by the time the load data is available, it is already in MEM/WB.

### Part 2: Load Data Forwarding Integration

The extension is already incorporated in Part 1 via `mem_wb_result`:

```systemverilog
logic [31:0] mem_wb_result;
assign mem_wb_result = mem_wb_mem_to_reg ? mem_wb_read_data : mem_wb_alu_result;
```

**Why this is correct:**
- If MEM/WB holds a load instruction: `mem_wb_mem_to_reg = 1`, so `mem_wb_result = mem_wb_read_data` (the value loaded from memory). The consumer receives the correct memory data.
- If MEM/WB holds an ALU instruction: `mem_wb_mem_to_reg = 0`, so `mem_wb_result = mem_wb_alu_result`. The consumer receives the computed value.

This handles the case: `LW x1, 0(x2)` followed by a stall, followed by `ADD x3, x1, x4`. After the stall:
- LW is in MEM/WB with `mem_wb_mem_to_reg=1` and `mem_wb_read_data = loaded_value`.
- ADD is in EX with `id_ex_rs1 = 1`.
- `mem_wb_rd = 1 == id_ex_rs1 = 1` -> `forward_a = 2'b01`.
- `alu_input_a = mem_wb_result = mem_wb_read_data = loaded_value`. Correct.

### Part 3: Forward MUX Trace for the Example Sequence

Recall: `ADD x1` enters EX in cycle 3, SUB x4 in cycle 4, AND x6 in cycle 5, OR x7 in cycle 6.

**When SUB x4, x1, x5 is in EX (cycle 4):**
```
EX/MEM holds: ADD x1 (just finished EX in cycle 3, now in MEM in cycle 4)
  ex_mem_reg_write = 1, ex_mem_rd = 1
  id_ex_rs1 = 1 (SUB reads x1)
  Condition: ex_mem_reg_write=1, ex_mem_rd=1 != 0, ex_mem_rd==id_ex_rs1 -> TRUE
  ForwardA = 2'b10 (EX/MEM forward)
  alu_input_a = ex_mem_alu_result (ADD's result)

id_ex_rs2 = 5 (SUB reads x5, no hazard)
  ForwardB = 2'b00 (no forward)
```

**When AND x6, x1, x4 is in EX (cycle 5):**
```
EX/MEM holds: SUB x4 (just finished EX in cycle 4)
  ex_mem_reg_write = 1, ex_mem_rd = 4
MEM/WB holds: ADD x1 (finished EX cycle 3, MEM cycle 4, now in WB cycle 5)
  mem_wb_reg_write = 1, mem_wb_rd = 1

id_ex_rs1 = 1 (AND reads x1):
  EX/MEM check: ex_mem_rd=4 != id_ex_rs1=1 -> FALSE
  MEM/WB check: mem_wb_rd=1 == id_ex_rs1=1 -> TRUE
  ForwardA = 2'b01 (MEM/WB forward for x1)

id_ex_rs2 = 4 (AND reads x4):
  EX/MEM check: ex_mem_rd=4 == id_ex_rs2=4 -> TRUE (override MEM/WB)
  ForwardB = 2'b10 (EX/MEM forward for x4 from SUB)
```

**When OR x7, x6, x8 is in EX (cycle 6):**
```
EX/MEM holds: AND x6 (just finished EX in cycle 5)
  ex_mem_reg_write = 1, ex_mem_rd = 6
MEM/WB holds: SUB x4 (finished EX cycle 4, MEM cycle 5, now in WB cycle 6)

id_ex_rs1 = 6 (OR reads x6):
  EX/MEM check: ex_mem_rd=6 == id_ex_rs1=6 -> TRUE
  ForwardA = 2'b10 (EX/MEM forward for x6)

id_ex_rs2 = 8 (OR reads x8, no hazard):
  ForwardB = 2'b00
```

**Summary table:**

| Instruction in EX | Cycle | ForwardA | ForwardA source | ForwardB | ForwardB source |
|-------------------|-------|----------|-----------------|----------|-----------------|
| ADD x1, x2, x3   | 3     | 00       | RF (x2)         | 00       | RF (x3)         |
| SUB x4, x1, x5   | 4     | 10       | EX/MEM (x1)     | 00       | RF (x5)         |
| AND x6, x1, x4   | 5     | 01       | MEM/WB (x1)     | 10       | EX/MEM (x4)     |
| OR  x7, x6, x8   | 6     | 10       | EX/MEM (x6)     | 00       | RF (x8)         |

### Part 4: Double-Hazard Priority Proof

```
I1: ADD x1, x2, x3  -- x1 = 10; enters EX cycle 3
I2: ADD x1, x4, x5  -- x1 = 20; enters EX cycle 4
I3: ADD x6, x1, x7  -- must use x1 = 20; enters EX cycle 5
```

**State when I3 is in EX (cycle 5):**
```
EX/MEM register: holds I2's result (I2 completed EX in cycle 4, now in MEM in cycle 5)
  ex_mem_reg_write = 1
  ex_mem_rd        = 1    (I2 writes x1)
  ex_mem_alu_result = 20  (I2's result: x1 = 20)

MEM/WB register: holds I1's result (I1 completed EX cycle 3, MEM cycle 4, now in WB cycle 5)
  mem_wb_reg_write = 1
  mem_wb_rd        = 1    (I1 writes x1)
  mem_wb_result    = 10   (I1's result: x1 = 10)

Consumer: I3 in EX, id_ex_rs1 = 1
```

**Forwarding logic evaluation (for ForwardA):**

```
Step 1: Check MEM/WB (lower priority):
  mem_wb_reg_write = 1    -- TRUE
  mem_wb_rd = 1 != 0      -- TRUE
  mem_wb_rd = 1 == id_ex_rs1 = 1  -- TRUE
  => ForwardA = 2'b01 (tentatively: use MEM/WB result = 10)

Step 2: Check EX/MEM (higher priority; overrides if true):
  ex_mem_reg_write = 1    -- TRUE
  ex_mem_rd = 1 != 0      -- TRUE
  ex_mem_rd = 1 == id_ex_rs1 = 1  -- TRUE
  => ForwardA = 2'b10 (override: use EX/MEM result = 20)

Final: ForwardA = 2'b10, alu_input_a = ex_mem_alu_result = 20
```

**Proof of correctness:**

I2 is more recent than I1 in program order. I2's write to x1 happens after I1's write. The architecturally correct value of x1 at the point I3 reads it is 20 (from I2). The EX/MEM register holds I2's result (the most recent write to x1 that has completed EX). The MEM/WB register holds I1's result (an older, stale write to x1). The priority logic correctly selects the EX/MEM value.

If the priority were reversed (MEM/WB had higher priority), I3 would receive x1 = 10 — the result of the older, stale instruction. This would be a silent data corruption bug with no exception, extremely difficult to diagnose in simulation.

---

## Discussion

### Where this forwarding unit does NOT apply

This forwarding unit covers register-to-ALU forwarding only. Several other forwarding paths exist in a complete pipeline:

1. **Store data forwarding:** The value being stored (`rs2` for SW/SH/SB) must also be forwarded if it is a RAW hazard. The forwarding MUX for the store data path is separate from the ALU input forwarding.

2. **Branch condition forwarding:** If branch comparison is done in EX (or ID), the branch's source registers may also need forwarding from EX/MEM or MEM/WB. The same forwarding conditions apply, but the MUX feeds the branch comparator rather than the ALU.

3. **WB-to-ID forwarding:** Some implementations add a direct forwarding path from the WB result to the ID stage (for the register file read), eliminating the need for MEM/WB -> EX forwarding in many cases. This requires the register file to support "read with forward" semantics.

### Testing the forwarding unit

A complete test suite should include:

```systemverilog
// Test case: double hazard (both EX/MEM and MEM/WB match rs1)
// Expected: ForwardA = 2'b10 (EX/MEM wins)
ex_mem_reg_write = 1; ex_mem_rd = 1; ex_mem_alu_result = 20;
mem_wb_reg_write = 1; mem_wb_rd = 1; mem_wb_result = 10;
id_ex_rs1 = 1;
// Assert: forward_a == 2'b10, alu_input_a == 20

// Test case: x0 write (must never forward)
ex_mem_reg_write = 1; ex_mem_rd = 0;  // writing x0
id_ex_rs1 = 0;
// Assert: forward_a == 2'b00 (no forward; x0 always reads as 0)

// Test case: load forwarding via MEM/WB
mem_wb_reg_write = 1; mem_wb_rd = 3; mem_wb_mem_to_reg = 1;
mem_wb_read_data = 0xDEADBEEF; mem_wb_alu_result = 0xBADC0DE;
id_ex_rs1 = 3;
// Assert: forward_a == 2'b01, alu_input_a == 0xDEADBEEF
```
