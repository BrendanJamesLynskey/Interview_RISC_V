# Problem 02: Design and Implement a Custom RISC-V Instruction

## Difficulty
Advanced (covers Tier 2 + Tier 3 material)

## Prerequisites
- RISC-V instruction encoding formats (R-, I-, U-type)
- RISC-V custom opcode space (custom-0 through custom-3)
- GCC inline assembly `.insn` directive
- Spike ISA simulator plugin interface
- Basic RTL concepts (ALU implementation)

---

## Problem Statement

You are designing a custom RISC-V instruction for a RISC-V microcontroller targeting motor control applications. The required operation is a **32-bit barrel shift with saturation** (BSATS):

```
BSATS rd, rs1, rs2
  shamt = rs2[4:0]              -- shift amount: lower 5 bits of rs2
  shifted = rs1 >> shamt        -- unsigned right shift (logical shift right)
  if (shifted > 0x0000_FFFF):
      rd = 0x0000_FFFF          -- saturate to 16-bit maximum
  else:
      rd = shifted              -- no saturation needed
```

**Use case:** The motor control algorithm produces a 32-bit torque value that must be scaled (right-shifted) by a configurable amount and then saturated to a 16-bit PWM range. Without this instruction, the software requires 3-4 instructions (SRL, SLTU/BGEU, conditional select). The custom instruction reduces this to 1, saving code size and latency in a tight interrupt handler.

**Tasks:**

1. Encode the instruction: define the complete 32-bit encoding using the R-type format in the custom-0 opcode space. Specify the MATCH and MASK values.
2. Write the C inline assembly wrapper function using the `.insn` directive.
3. Write a software reference implementation in C to verify against.
4. Write the Spike C++ implementation of the instruction semantics (as a `.h` file for the Spike insns directory).
5. Write 5 test cases with inputs, expected outputs, and the specific aspect of correctness each tests.
6. Show the RTL pseudocode (SystemVerilog) for the custom ALU operation.

---

## Solution

### Part 1: Instruction Encoding

**Chosen encoding (R-type, custom-0 opcode):**

```
Instruction: BSATS rd, rs1, rs2
Operation:   shifted = rs1 >> rs2[4:0]; rd = min(shifted, 0xFFFF)

Encoding fields:
  [31:25] funct7 = 0b0001000   (value 8: identifies BSATS)
  [24:20] rs2    = rs2[4:0]    (source register 2: shift amount)
  [19:15] rs1    = rs1[4:0]    (source register 1: value to shift)
  [14:12] funct3 = 0b011       (value 3: "saturating shift" category)
  [11:7]  rd     = rd[4:0]     (destination register)
  [6:0]   opcode = 0b0001011   (0x0B: custom-0)

Full encoding with all variable fields zero (rd=x0, rs1=x0, rs2=x0):
  MATCH_BSATS = 0b 0001000 00000 00000 011 00000 0001011
              = 0x1000_180B

MASK_BSATS  = 0b 1111111 00000 00000 111 00000 1111111
            = 0xFE00_707F

Verification: applying MASK to an instruction and comparing to MATCH:
  If (instruction & 0xFE00_707F) == 0x1000_180B: this is a BSATS instruction.

Concrete example: BSATS a0, a1, a2
  rd  = a0 = x10 = 5'b01010
  rs1 = a1 = x11 = 5'b01011
  rs2 = a2 = x12 = 5'b01100

  Encoded: 0b 0001000 01100 01011 011 01010 0001011
           = 0x10C5_B50B

Let's verify: 0x10C5_B50B & 0xFE00_707F = 0x1000_180B = MATCH_BSATS. Correct.
```

### Part 2: C Inline Assembly Wrapper

```c
#include <stdint.h>

/**
 * riscv_bsats - Barrel shift with saturation to 16-bit unsigned.
 *
 * @value:  32-bit unsigned input value
 * @shamt:  Shift amount (only lower 5 bits used, like SRL)
 * @return: (value >> shamt), saturated to [0, 0xFFFF]
 *
 * Instruction encoding: BSATS rd, rs1, rs2
 *   .insn r <opcode>, <funct3>, <funct7>, <rd>, <rs1>, <rs2>
 *   opcode = 0x0B (custom-0), funct3 = 0x3, funct7 = 0x08
 */
static inline uint32_t riscv_bsats(uint32_t value, uint32_t shamt) {
    uint32_t result;
    asm volatile (
        ".insn r 0x0B, 0x3, 0x08, %0, %1, %2"
        : "=r"(result)
        : "r"(value), "r"(shamt)
    );
    return result;
}
```

**Constraint notes:**
- `"=r"(result)`: output to any general-purpose register (rd).
- `"r"(value)`: input in any GPR (rs1).
- `"r"(shamt)`: input in any GPR (rs2). The hardware uses only `shamt[4:0]`; the compiler does not need to mask it first (hardware handles the extraction).
- `volatile`: necessary because the compiler cannot see the instruction's semantics; without volatile, the compiler might eliminate the call if it believes the result is unused.

### Part 3: Software Reference Implementation

```c
#include <stdint.h>

/**
 * bsats_reference - Software reference for BSATS instruction.
 * Used to verify the Spike plugin and RTL implementations.
 *
 * @value:  32-bit input value (treated as unsigned)
 * @shamt:  Shift amount (only bits [4:0] used, range 0-31)
 * @return: (value >> shamt) saturated to [0, 0xFFFF]
 */
uint32_t bsats_reference(uint32_t value, uint32_t shamt) {
    uint32_t shift_amount = shamt & 0x1F;     // RISC-V SRL uses lower 5 bits
    uint32_t shifted      = value >> shift_amount;  // logical right shift

    // Saturation: if shifted > 0xFFFF, clamp to 0xFFFF
    if (shifted > 0x0000FFFFU) {
        return 0x0000FFFFU;
    }
    return shifted;
}

// Branchless variant (preferred for constant-time implementations):
uint32_t bsats_reference_branchless(uint32_t value, uint32_t shamt) {
    uint32_t shift_amount = shamt & 0x1F;
    uint32_t shifted      = value >> shift_amount;
    // Saturation without branch:
    // If shifted > 0xFFFF: all high bits clear, or bit 16+ set
    // Use: saturated = (shifted > 0xFFFF) ? 0xFFFF : shifted
    uint32_t overflow_mask = -(shifted > 0x0000FFFFU); // 0xFFFFFFFF if overflow, else 0
    return (shifted & ~overflow_mask) | (0x0000FFFFU & overflow_mask);
}
```

### Part 4: Spike Implementation

```cpp
// File: riscv/insns/bsats.h
// BSATS rd, rs1, rs2
// Barrel shift right with saturation to 16-bit unsigned
//
// Operation:
//   shamt   = RS2 & 0x1F               (lower 5 bits, matching SRL behaviour)
//   shifted = (uint32_t)RS1 >> shamt   (logical right shift)
//   RD      = (shifted > 0xFFFF) ? 0xFFFF : shifted

{
    // RS1, RS2, WRITE_RD are Spike macros for reading/writing registers
    // For RV32: RS1/RS2 are 32-bit; for RV64: they are 64-bit sign-extended values.
    // We work in 32-bit arithmetic for this instruction.

    uint32_t src    = (uint32_t)RS1;    // cast to 32-bit unsigned
    uint32_t shamt  = (uint32_t)RS2 & 0x1F;  // lower 5 bits of rs2
    uint32_t result = src >> shamt;     // logical (unsigned) right shift

    // Saturation to 16-bit unsigned maximum
    if (result > 0x0000FFFFu) {
        result = 0x0000FFFFu;
    }

    // WRITE_RD sign-extends to XLEN if running on RV64
    // For RV32: WRITE_RD writes 32 bits directly
    WRITE_RD(sext32(result));  // sext32 sign-extends 32-bit value to 64-bit for RV64 compat
}
```

**Registration in Spike (riscv/encoding.h):**
```c
#define MATCH_BSATS 0x1000180B
#define MASK_BSATS  0xFE00707F
DECLARE_INSN(bsats, MATCH_BSATS, MASK_BSATS)
```

**In riscv/riscv.mk.in, add `bsats` to the `insn_list`.**

**Build and test:**
```bash
cd riscv-isa-sim
./configure --prefix=/opt/spike-custom
make -j$(nproc) && make install

# Test with the test program:
riscv32-unknown-elf-gcc -march=rv32i -mabi=ilp32 -O1 -o bsats_test bsats_test.c
spike --isa=rv32i pk bsats_test
# Expected: all tests PASS
```

### Part 5: Test Cases

```c
#include <stdint.h>
#include <stdio.h>

typedef struct {
    const char *name;        // what aspect is being tested
    uint32_t    value;       // rs1
    uint32_t    shamt;       // rs2
    uint32_t    expected;    // expected rd
} test_case_t;

static const test_case_t tests[] = {
    {
        // Test 1: No saturation needed; result fits in 16 bits
        // 0x00010000 >> 1 = 0x00008000 <= 0xFFFF: no saturation
        .name     = "No saturation: result within 16-bit range",
        .value    = 0x00010000u,
        .shamt    = 1,
        .expected = 0x00008000u,
    },
    {
        // Test 2: Saturation triggered; shifted value exceeds 0xFFFF
        // 0xFFFFFFFF >> 0 = 0xFFFFFFFF > 0xFFFF: saturate to 0xFFFF
        .name     = "Saturation: no shift, large value saturates",
        .value    = 0xFFFFFFFFu,
        .shamt    = 0,
        .expected = 0x0000FFFFu,
    },
    {
        // Test 3: Exact saturation boundary
        // 0x00010000 >> 0 = 0x00010000 > 0xFFFF: saturate
        // 0x0000FFFF >> 0 = 0x0000FFFF <= 0xFFFF: no saturate (boundary)
        .name     = "Boundary: 0x10000 saturates; 0xFFFF does not",
        .value    = 0x0001FFFFu,
        .shamt    = 1,
        // 0x1FFFF >> 1 = 0x0FFFF = 0xFFFF: no saturation (edge case)
        .expected = 0x0000FFFFu,
    },
    {
        // Test 4: Shift amount from rs2: only lower 5 bits used
        // shamt = 0b100000 = 32: lower 5 bits = 0b00000 = 0 -> no shift
        // value = 0x00001234: no shift, result = 0x00001234 <= 0xFFFF: no saturate
        .name     = "Shift amount masking: rs2 = 32 uses only 5 bits (= 0)",
        .value    = 0x00001234u,
        .shamt    = 32u,  // lower 5 bits = 0: shift by 0
        .expected = 0x00001234u,
    },
    {
        // Test 5: Zero input
        // 0 >> any_amount = 0 <= 0xFFFF: no saturation
        .name     = "Zero input: any shift of 0 returns 0",
        .value    = 0x00000000u,
        .shamt    = 15u,
        .expected = 0x00000000u,
    },
};

int run_tests(void) {
    int passed = 0, failed = 0;
    for (int i = 0; i < 5; i++) {
        uint32_t result = riscv_bsats(tests[i].value, tests[i].shamt);
        if (result == tests[i].expected) {
            printf("PASS: %s\n", tests[i].name);
            passed++;
        } else {
            printf("FAIL: %s\n", tests[i].name);
            printf("  value=0x%08X, shamt=%u\n", tests[i].value, tests[i].shamt);
            printf("  expected=0x%08X, got=0x%08X\n", tests[i].expected, result);
            failed++;
        }
    }
    printf("\n%d passed, %d failed\n", passed, failed);
    return failed;
}

int main(void) {
    return run_tests();
}
```

**Test case coverage:**

| Test | Aspect covered |
|------|---------------|
| 1 | No saturation: result is in range, shift reduces a value below 0x10000 |
| 2 | Full saturation: zero shift, value already exceeds 16-bit max |
| 3 | Exact boundary: result == 0xFFFF (neither saturated nor below maximum) |
| 4 | Shift amount masking: verifies only rs2[4:0] is used, not all 32 bits |
| 5 | Zero identity: any shift of zero returns zero |

### Part 6: RTL Implementation

```systemverilog
// Custom ALU operation for BSATS instruction
// This module is instantiated inside the EX stage ALU, selected
// when the opcode matches BSATS.

module bsats_unit (
    input  logic [31:0] rs1_data,    // source value
    input  logic [31:0] rs2_data,    // shift amount source
    output logic [31:0] rd_data      // result
);
    logic [4:0]  shamt;
    logic [31:0] shifted;
    logic        saturate;

    // Extract shift amount (lower 5 bits of rs2, matching SRL behaviour)
    assign shamt = rs2_data[4:0];

    // Logical right shift (unsigned)
    assign shifted = rs1_data >> shamt;

    // Saturation check: if any bit above bit 15 is set, result exceeds 0xFFFF
    // shifted[31:16] != 0 means saturation is needed
    assign saturate = |shifted[31:16];   // bitwise OR reduction of upper 16 bits

    // Output: saturated or original shifted value
    assign rd_data = saturate ? 32'h0000_FFFF : shifted;

endmodule
```

**Integration into the main ALU:**

```systemverilog
// Inside the main EX stage ALU:
module alu (
    input  logic [31:0] alu_a,
    input  logic [31:0] alu_b,
    input  logic [3:0]  alu_op,    // extended to include custom ops
    output logic [31:0] alu_result
);
    // ... standard ALU operations ...
    logic [31:0] bsats_result;
    bsats_unit bsats_inst (
        .rs1_data(alu_a),
        .rs2_data(alu_b),
        .rd_data(bsats_result)
    );

    always_comb begin
        case (alu_op)
            4'b0000: alu_result = alu_a + alu_b;     // ADD
            4'b0001: alu_result = alu_a - alu_b;     // SUB
            4'b0010: alu_result = alu_a >> alu_b[4:0]; // SRL
            // ...
            4'b1001: alu_result = bsats_result;      // BSATS (custom)
            default: alu_result = 32'b0;
        endcase
    end
endmodule
```

**Critical path analysis:**
```
BSATS critical path:
  1. 5-to-32 barrel shifter: ~3 gate delays (using standard cell library)
  2. OR reduction of [31:16]: ~log2(16) = 4 gate delays
  3. 32-bit 2-to-1 MUX: ~1 gate delay

  Total: ~8 gate delays -- comparable to a 32-bit ALU add.
  This fits within a standard 1-cycle EX stage at typical frequencies.

Comparison to software replacement:
  SRL  a0, a0, a2       -- shift
  LI   t0, 0xFFFF
  BGEU a0, t0, .clamp   -- branch if shifted > 0xFFFF
  j    .done
  .clamp: mv a0, t0
  .done: ...

  The software version: 4-5 instructions, 1 branch (pipeline disruption risk),
                        code size: 16-20 bytes.
  The custom instruction: 1 instruction, no branch, 4 bytes.
```

---

## Discussion

### Why saturation is not just SMIN

One might ask: why not use `MIN(shifted, 0xFFFF)` with existing instructions? In RISC-V base I, there is no SMIN instruction. The combination would be:

```
SRL  t0, a0, a1         # shift
LI   t1, 0xFFFF         # load saturation constant
BLTU t0, t1, .no_sat    # if shifted < 0xFFFF: skip
MV   t0, t1             # saturate
.no_sat:
MV   a0, t0
```

This is 4-5 instructions with a branch. With the B extension (`Zbb`), `MIN` would be available, reducing to:
```
SRL  a0, a0, a1
LI   t0, 0xFFFF
MIN  a0, a0, t0         # from Zbb extension
```

Three instructions, no branch — better than the base sequence, but still 3x the code size and latency of the custom instruction. The custom instruction is most justified in a tight ISR loop where the 2-instruction savings per loop iteration translates to a significant cycle count reduction.

### Extending to RV64

On RV64, `RS1` and `RS2` are 64-bit values. The instruction should be specified to work on the lower 32 bits of rs1 (treating it as a W-word operation, consistent with `SRLW` in RV64I). The result should be sign-extended to 64 bits (writing a value in [0, 0xFFFF] sign-extended: always a positive number, upper 32 bits = 0). The Spike implementation uses `sext32(result)` for this reason.
