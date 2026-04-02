# Custom Extensions

## Prerequisites
- RISC-V instruction formats: R-, I-, S-, B-, U-, J-type encoding
- RISC-V opcode map and standard opcode assignments
- GCC inline assembly syntax (`asm volatile`)
- Basic understanding of compiler intrinsics and code generation

---

## Concept Reference

### Why Custom Instructions Exist

The RISC-V ISA is explicitly designed to be extensible. Unlike fixed ISAs, RISC-V reserves opcode space specifically for implementation-defined custom instructions. This allows hardware designers to accelerate domain-specific operations without breaking ISA compatibility or waiting for a new standard extension.

**Common motivations:**
- Accelerate cryptographic operations (AES round function in a single instruction).
- Add domain-specific compute: DSP MAC (multiply-accumulate), CRC computation, bitfield manipulation.
- Reduce code size by combining common multi-instruction idioms.
- Enable hardware-software co-design for custom accelerators (e.g., a neural network inference instruction).

### Reserved Custom Opcode Space

The RISC-V base ISA reserves four opcode values for custom 32-bit instructions (and additional space for custom 16-bit compressed instructions):

```
32-bit custom instruction opcodes (bits [6:0]):
  0x0B (0b000_1011) -- custom-0: intended for custom RV32 instructions
  0x2B (0b010_1011) -- custom-1: intended for custom RV64 instructions
  0x5B (0b101_1011) -- custom-2: reserved for custom or 48-bit+ encoding
  0x7B (0b111_1011) -- custom-3: reserved for custom or 64-bit+ encoding

Note on the overlapping standard opcodes:
  0x0B is NOT used by any standard extension.
  0x2B is NOT used by any standard extension.
  0x5B is NOT used by any standard extension.
  0x7B is used by RISC-V debug (EBREAK in debug mode uses a custom variant).
  In practice, custom-0 (0x0B) and custom-1 (0x2B) are the safest choices.
```

**16-bit (compressed) custom space:**
Quadrant 0 (bits [1:0] = 2'b00) with `funct3 = 3'b101` or `3'b110` or `3'b111` is reserved for custom instructions. Quadrant 1 and 2 have additional reserved ranges. These are rarely used in practice because the 16-bit encoding space is tight.

### Encoding a Custom 32-Bit Instruction

Custom instructions use the standard RISC-V 32-bit instruction format. You are free to define the upper 25 bits (bits [31:7]) in any way you choose, as long as bits [6:0] are one of the four custom opcodes and bits [1:0] are `11` (marking this as a 32-bit instruction).

**Most common approach: reuse the R-type encoding**

```
R-type custom instruction layout:
  [31:25] funct7   (7 bits) -- distinguishes different custom operations
  [24:20] rs2      (5 bits) -- source register 2 (or additional opcode bits)
  [19:15] rs1      (5 bits) -- source register 1
  [14:12] funct3   (3 bits) -- further opcode differentiation
  [11:7]  rd       (5 bits) -- destination register
  [6:0]   opcode   (7 bits) -- must be 0x0B, 0x2B, 0x5B, or 0x7B

With funct7 (7 bits) + funct3 (3 bits) = 10 bits of opcode differentiation,
you can define up to 1024 distinct custom R-type instructions per custom opcode.
```

**Example: SATURATE_ADD (signed 32-bit saturating addition)**
```
Encoding:
  funct7 = 0b0000001
  funct3 = 0b000
  opcode = 0x0B (custom-0)

Full encoding: {0b0000001, rs2[4:0], rs1[4:0], 0b000, rd[4:0], 0b0001011}

Assembly syntax (user-defined mnemonic):  sadd rd, rs1, rs2
RTL semantics:
  sum = signed(rs1) + signed(rs2)
  if sum > INT32_MAX: rd = INT32_MAX
  if sum < INT32_MIN: rd = INT32_MIN
  else: rd = sum
```

**Custom I-type instruction (immediate operand):**
```
I-type custom instruction layout:
  [31:20] imm[11:0]  (12 bits) -- immediate
  [19:15] rs1        (5 bits)
  [14:12] funct3     (3 bits)
  [11:7]  rd         (5 bits)
  [6:0]   opcode     (7 bits)

Example: BREV rd, rs1  (bit-reverse: no immediate, but if desired, bit width could be imm)
```

### Adding Custom Instructions to GCC

There are three main approaches, in increasing complexity:

**Approach 1: Inline assembly with `.insn` directive**
The GAS (GNU Assembler) `.insn` directive encodes arbitrary instruction words:

```c
// Inline assembly for SADD using .insn r format
static inline int32_t sadd(int32_t a, int32_t b) {
    int32_t result;
    asm volatile (
        ".insn r 0x0B, 0x0, 0x01, %0, %1, %2"
        : "=r"(result)     // output: rd
        : "r"(a), "r"(b)   // inputs: rs1, rs2
    );
    return result;
}

// .insn r <opcode>, <funct3>, <funct7>, <rd>, <rs1>, <rs2>
// Generates: {funct7=0x01, rs2, rs1, funct3=0x0, rd, opcode=0x0B}
```

**Approach 2: Custom compiler intrinsic with a GCC plugin or LLVM target**
For production use, define a `__builtin_riscv_sadd(a, b)` intrinsic. This requires either a GCC plugin (using GIMPLE/RTL IR hooks) or a custom LLVM backend target description. The compiler then generates the custom instruction directly from C, enabling constant propagation and other optimisations.

**Approach 3: Custom assembler mnemonic via binutils port**
Modify `gas/config/tc-riscv.c` and the `include/opcode/riscv-opc.h` tables to add the new mnemonic. After rebuilding binutils, you can write `sadd a0, a1, a2` in RISC-V assembly files and have it assembled to the correct encoding.

```c
// In include/opcode/riscv-opc.h:
#define MATCH_SADD 0x0200000B   // funct7=1, funct3=0, opcode=custom-0
#define MASK_SADD  0xFE00707F

// In opcodes/riscv-opc.c:
{"sadd", 0, INSN_CLASS_I, "d,s,t", MATCH_SADD, MASK_SADD, match_opcode, 0},
//  mnemonic, ISA_class, operand_format, match_bits, mask_bits, match_fn, attributes
```

### Toolchain Integration: LLVM

LLVM's RISC-V backend is modular and well-suited for adding custom instructions. The key file is `llvm/lib/Target/RISCV/RISCVInstrInfo.td`:

```tablegen
// TableGen definition for a custom SADD instruction in LLVM:
def SADD : RVInst<(outs GPR:$rd), (ins GPR:$rs1, GPR:$rs2),
                  "sadd", "$rd, $rs1, $rs2",
                  [(set GPR:$rd, (int_riscv_sadd GPR:$rs1, GPR:$rs2))]> {
    bits<5> rs2;
    bits<5> rs1;
    bits<5> rd;

    let Inst{31-25} = 0b0000001;  // funct7
    let Inst{24-20} = rs2;
    let Inst{19-15} = rs1;
    let Inst{14-12} = 0b000;      // funct3
    let Inst{11-7}  = rd;
    let Inst{6-0}   = 0b0001011;  // custom-0 opcode
}

// Corresponding intrinsic declaration:
def int_riscv_sadd : Intrinsic<[llvm_i32_ty],
                                [llvm_i32_ty, llvm_i32_ty],
                                [IntrNoMem, IntrSpeculatable, IntrWillReturn]>;
```

**C-level usage after LLVM integration:**
```c
// Declare the compiler intrinsic
int32_t __builtin_riscv_sadd(int32_t a, int32_t b);

// Use in application code
int32_t result = __builtin_riscv_sadd(x, y);
// Compiler emits: sadd result_reg, x_reg, y_reg
```

### Testing and Verification

A custom instruction needs to be verified at multiple levels:

```
Verification pyramid for custom instructions:

1. Functional model (Spike plugin or software reference):
   - Verify the instruction's semantic with a C++ reference implementation.
   - Run compliance-style tests: known inputs -> expected outputs.

2. RTL unit test:
   - Simulate the custom execution unit in isolation (ALU block test).
   - Cover all operand combinations, including boundary values.

3. Pipeline integration test:
   - Verify forwarding paths: does a result produced by the custom instruction
     forward correctly to a dependent instruction one cycle later?
   - Verify load-use interaction: does the custom instruction stall correctly
     if its operand is produced by a load?

4. Co-simulation with Spike:
   - Run random instruction streams including the custom instruction.
   - Compare RTL output against Spike plugin reference.

5. Toolchain integration test:
   - Compile a known benchmark using the custom instruction via intrinsics.
   - Verify the output binary executes correctly and produces expected results.
```

---

## Tier 1 — Fundamentals

### Question F1
**What opcode space does the RISC-V ISA reserve for custom 32-bit instructions? Why were these specific opcodes chosen, and what constraints must a custom instruction encoding obey?**

**Answer:**

The RISC-V ISA reserves four 7-bit opcodes for custom 32-bit instructions:
- `0x0B` (custom-0)
- `0x2B` (custom-1)
- `0x5B` (custom-2)
- `0x7B` (custom-3)

**Why these specific opcodes:** The RISC-V opcode map is organised so that bits [6:0] of a 32-bit instruction indicate the instruction length and major category. Bits [1:0] = `11` identifies a 32-bit instruction. Within the 32-bit space, the upper 5 bits ([6:2]) identify the major instruction type. The four custom opcodes were chosen to not conflict with any base ISA or standard extension instructions, leaving them permanently available for implementation-defined use.

**Constraints on custom instruction encodings:**

1. **Bits [1:0] must be `11`:** This is mandatory for any 32-bit RISC-V instruction. It tells the decoder this is a 32-bit (not 16-bit compressed) instruction. Violating this would cause a 16-bit decoder to misparse the instruction.

2. **Bits [6:2] must match the chosen custom opcode:** i.e., bits [6:0] must be exactly `0x0B`, `0x2B`, `0x5B`, or `0x7B`.

3. **No conflict with standard extensions:** If the processor also implements a standard extension that uses the same custom opcode range for its own encoding, there is a conflict. In practice, custom-0 and custom-1 are confirmed to be free of all ratified standard extensions. Custom-2 and custom-3 are also currently free but were at one point reserved for possible longer-instruction encodings.

4. **Self-consistency:** The encoding must be deterministic — given the same instruction bits, the hardware always produces the same result. The encoding can use any of the standard field positions (rd, rs1, rs2, funct3, funct7, immediate) or invent entirely new field layouts.

---

### Question F2
**Write the inline assembly C code to invoke a custom RISC-V instruction with encoding `funct7=0b0000010, funct3=0b001, opcode=custom-0` that computes `rd = CRC32(rs1, rs2)`. Show the `.insn` directive syntax and the correct constraint strings.**

**Answer:**

```c
#include <stdint.h>

// Custom instruction: CRC32 rd, rs1, rs2
// Encoding: funct7=0x02, funct3=0x1, opcode=0x0B (custom-0)
// Semantics: rd = CRC32 of (rs1 interpreted as data, rs2 as polynomial)

static inline uint32_t riscv_crc32(uint32_t data, uint32_t poly) {
    uint32_t result;
    asm volatile (
        // .insn r <opcode>, <funct3>, <funct7>, <rd>, <rs1>, <rs2>
        ".insn r 0x0B, 0x1, 0x02, %0, %1, %2"
        : "=r"(result)       // output operand: rd (any integer register)
        : "r"(data),         // input operand: rs1
          "r"(poly)          // input operand: rs2
        :                    // no clobbers (instruction only writes rd)
    );
    return result;
}
```

**Explanation of constraints:**

- `"=r"(result)`: `=` means write-only output; `r` means any general-purpose register. The compiler assigns a register to `rd` and maps it to `result` after the instruction.
- `"r"(data)`: `r` means any general-purpose register. The compiler loads `data` into a register and assigns it to `rs1`.
- `"r"(poly)`: similarly for `rs2`.
- The volatile qualifier: prevents the compiler from eliminating or reordering the asm block. Required because the compiler cannot see inside the custom instruction's semantics.

**Verification of the encoding:**
```
.insn r 0x0B, 0x1, 0x02, a0, a1, a2
Generates:
  [31:25] = 0x02    = 0b000_0010
  [24:20] = a2 = x12 = 5'b01100
  [19:15] = a1 = x11 = 5'b01011
  [14:12] = 0x1     = 0b001
  [11:7]  = a0 = x10 = 5'b01010
  [6:0]   = 0x0B    = 0b000_1011

32-bit word: 0000_010_01100_01011_001_01010_0001011
           = 0x04C5_B50B
```

---

### Question F3
**What is the difference between a standard RISC-V extension (like Zba or M) and a custom extension? Under what conditions would you choose a custom extension over waiting for a standard one to be ratified?**

**Answer:**

**Standard extension:**
- Formally specified by RISC-V International and ratified through the community process.
- Assigned a standard ISA string (e.g., `_zba`, `m`, `v`).
- Implemented identically across all conformant RISC-V processors.
- Supported by mainstream toolchains (GCC, LLVM, binutils) without modification.
- Guaranteed not to conflict with other standard extensions.
- Binary compatibility: a binary using a standard extension runs on any processor that implements that extension.

**Custom extension:**
- Defined by the hardware implementer with no external oversight.
- Uses one of the four reserved custom opcode spaces.
- Not portable: binaries using custom instructions only run on processors that implement those specific customs.
- Requires toolchain modifications (custom GAS mnemonic, compiler intrinsic, or `.insn` inline asm).
- No guarantee of non-conflict with other vendors' custom extensions using the same opcode bits.

**When to choose a custom extension:**

1. **Time to market:** Standard extension ratification takes years. If the design has a critical deadline and the needed instruction is clearly beneficial (e.g., AES acceleration for a security chip), a custom extension is faster.

2. **Proprietary IP:** The operation is a trade secret (e.g., a proprietary ML inference kernel) and publishing a standard extension would disclose IP.

3. **Highly specific use case:** The operation is so specialised that it would never be adopted as a standard (e.g., a specific filter coefficient computation for one audio codec IC). The standardisation process is not worthwhile.

4. **Experimental / research:** Exploring whether a candidate instruction is beneficial before investing in the standardisation process. Many instructions start as custom and later become standard proposals (e.g., some Zk cryptography instructions began as vendor customs).

5. **Single-vendor ecosystem:** The SoC will only run software built specifically for it (embedded product with no third-party software). Binary portability is not a concern.

**When to prefer a standard extension:** When software portability is important, when using third-party open-source software that is already compiled with standard extension support, or when the operation (like bit manipulation or packed SIMD) is broadly useful and a standard exists or is close to ratification.

---

## Tier 2 — Intermediate

### Question I1
**Design the complete encoding, assembly syntax, and C intrinsic for a custom RISC-V instruction `VDOT8` that computes the 4-element 8-bit dot product: `rd = sum(rs1[7:0]*rs2[7:0], rs1[15:8]*rs2[15:8], rs1[23:16]*rs2[23:16], rs1[31:24]*rs2[31:24])` for RV32 (result is a 32-bit integer). Include overflow handling in the specification.**

**Answer:**

**Semantics specification:**
```
VDOT8 rd, rs1, rs2

Let:
  a[i] = signed byte i of rs1: rs1[(i*8+7):(i*8)] for i in {0,1,2,3}
  b[i] = signed byte i of rs2: rs2[(i*8+7):(i*8)] for i in {0,1,2,3}

Operation:
  result = a[0]*b[0] + a[1]*b[1] + a[2]*b[2] + a[3]*b[3]

Each product a[i]*b[i] is a signed 8-bit x 8-bit multiply:
  range: [-128 * 128, 127 * 127] = [-16384, 16129]
  fits in a signed 16-bit value.

Sum of four such products:
  range: [4 * -16384, 4 * 16129] = [-65536, 64516]
  fits in a signed 32-bit value without overflow.
  rd is a signed 32-bit result.

Note: no overflow is possible for this operation (the maximum absolute sum is
      4 * 128 * 128 = 65536 which fits in 17 bits). rd always holds the exact result.
```

**Instruction encoding (R-type, custom-0 opcode):**
```
[31:25] funct7 = 0b0000011   (value 3; distinguishes VDOT8 from other customs)
[24:20] rs2    = rs2 index   (5 bits)
[19:15] rs1    = rs1 index   (5 bits)
[14:12] funct3 = 0b010       (value 2; could indicate "byte SIMD dot product")
[11:7]  rd     = rd index    (5 bits)
[6:0]   opcode = 0b0001011   (0x0B, custom-0)

MATCH_VDOT8 = 0x0600_2_00B    -- assembled: funct7=3, funct3=2, opcode=0x0B
            = 0b 0000011 00000 00000 010 00000 0001011
            = 0x0600_200B

MASK_VDOT8  = 0xFE00_707F    -- covers funct7, funct3, opcode; ignores rs1, rs2, rd
```

**GAS binutils entry:**
```c
// In riscv-opc.h:
#define MATCH_VDOT8  0x0600200B
#define MASK_VDOT8   0xFE00707F

// In riscv-opc.c:
{"vdot8", 0, INSN_CLASS_I, "d,s,t", MATCH_VDOT8, MASK_VDOT8, match_opcode, 0},
// "d,s,t" = operand format: rd, rs1, rs2
```

**C inline assembly intrinsic:**
```c
#include <stdint.h>

// VDOT8: signed 4-element 8-bit dot product
// rs1 and rs2 are packed with 4 signed bytes each (little-endian byte ordering)
static inline int32_t riscv_vdot8(int32_t rs1, int32_t rs2) {
    int32_t result;
    asm volatile (
        ".insn r 0x0B, 0x2, 0x03, %0, %1, %2"
        : "=r"(result)
        : "r"(rs1), "r"(rs2)
    );
    return result;
}

// Helper to pack 4 bytes into the operand format
static inline int32_t pack_bytes(int8_t b0, int8_t b1, int8_t b2, int8_t b3) {
    return ((uint32_t)(uint8_t)b0)        |
           ((uint32_t)(uint8_t)b1 << 8)   |
           ((uint32_t)(uint8_t)b2 << 16)  |
           ((uint32_t)(uint8_t)b3 << 24);
}
```

**Usage example:**
```c
int main(void) {
    // Dot product of {1, 2, 3, 4} . {5, 6, 7, 8}
    // = 1*5 + 2*6 + 3*7 + 4*8 = 5 + 12 + 21 + 32 = 70
    int32_t a = pack_bytes(1, 2, 3, 4);
    int32_t b = pack_bytes(5, 6, 7, 8);
    int32_t result = riscv_vdot8(a, b);  // should return 70
    return (result == 70) ? 0 : 1;       // 0 = pass, 1 = fail
}
```

**LLVM TableGen definition:**
```tablegen
def VDOT8 : RVInst<(outs GPR:$rd), (ins GPR:$rs1, GPR:$rs2),
                   "vdot8", "$rd, $rs1, $rs2",
                   [(set GPR:$rd, (int_riscv_vdot8 GPR:$rs1, GPR:$rs2))]> {
    bits<5> rs2;
    bits<5> rs1;
    bits<5> rd;
    let Inst{31-25} = 0b0000011;  // funct7 = 3
    let Inst{24-20} = rs2;
    let Inst{19-15} = rs1;
    let Inst{14-12} = 0b010;      // funct3 = 2
    let Inst{11-7}  = rd;
    let Inst{6-0}   = 0b0001011;  // custom-0
}
```

---

### Question I2
**A RISC-V custom extension is designed for a cryptographic accelerator. The custom instruction `AESENC rd, rs1, rs2` performs one AES round. Describe the potential pipeline integration hazards introduced by this instruction if it has a 4-cycle execution latency, and explain how the hardware stall logic must be modified.**

**Answer:**

**AES round instruction characteristics:**
- Reads two source registers (rs1 = state, rs2 = round key).
- Writes one destination register (rd = state after one AES round).
- Execution latency: 4 cycles (the AES unit is a 4-stage pipeline).
- This is a multi-cycle non-pipelined delay if the unit is not fully pipelined, or a 4-cycle latency if the unit is pipelined but occupies 4 EX stages.

**Pipeline hazards introduced:**

**1. RAW hazard with extended latency:**
In a standard 5-stage pipeline, an ALU instruction's result is available after 1 cycle in EX and can be forwarded from EX/MEM. An AESENC result is not available until 4 cycles after it enters EX. A dependent instruction 1, 2, or 3 instructions later cannot receive the result via standard forwarding.

```
Example:
  AESENC a0, a1, a2   -- result available at end of cycle EX+4
  XOR    a3, a0, a4   -- needs a0 at start of its EX
  (if XOR follows AESENC by 1 instruction: 3 stall cycles needed)
  (if XOR follows AESENC by 2 instructions: 2 stall cycles needed)
  (if XOR follows AESENC by 3 instructions: 1 stall cycle needed)
  (if XOR follows AESENC by 4+ instructions: no stall)
```

**2. Structural hazard on the AES unit:**
If the AES unit is not fully pipelined (i.e., a second AESENC cannot enter the unit until the first has completed), two consecutive AESENC instructions cause a structural hazard. The second AESENC must stall for 4 cycles.

```
AESENC a0, a1, a2   -- occupies AES unit for cycles 3-6
AESENC a5, a0, a6   -- cannot enter AES unit until cycle 7; stall 4 cycles
                       (also a RAW hazard on a0)
```

If the AES unit is fully pipelined (a new instruction enters every cycle), structural hazards on the unit itself are eliminated, but the 4-cycle latency RAW hazard remains.

**3. Write-back port conflict (WAW risk):**
If two AESENC instructions complete WB in the same cycle (possible if they both enter EX in close succession and both have 4-cycle latency), they would both need to write to the register file in the same cycle. The WB stage's single write port becomes a structural hazard. This is handled by the stall logic: ensuring only one instruction writes per cycle.

**Modifications to the stall/forwarding logic:**

The standard hazard detection unit only checks 1 or 2 cycles of pipeline distance. For a 4-cycle custom instruction, the hazard detection must be extended:

```systemverilog
// Extended hazard detection for multi-cycle custom instruction
module hazard_detection_extended (
    // Standard inputs:
    input  logic        id_ex_mem_read,
    input  logic [4:0]  id_ex_rd,
    // Extended: custom instruction tracking
    input  logic        id_ex_is_aesenc,    // AESENC is in EX stage
    input  logic        ex_mem_is_aesenc,   // AESENC is in MEM stage
    input  logic        mem_wb_is_aesenc,   // AESENC is in WB stage (still executing)
    // (add a 4th stage if needed: wb_stage_is_aesenc)
    input  logic [4:0]  id_ex_aesenc_rd,
    input  logic [4:0]  ex_mem_aesenc_rd,
    input  logic [4:0]  mem_wb_aesenc_rd,
    input  logic [4:0]  if_id_rs1,
    input  logic [4:0]  if_id_rs2,
    output logic        stall
);
    // Standard load-use check
    logic load_use;
    assign load_use = id_ex_mem_read && (id_ex_rd != 5'b0) &&
                      ((id_ex_rd == if_id_rs1) || (id_ex_rd == if_id_rs2));

    // AESENC RAW checks: stall if result not yet available
    logic aesenc_hazard;
    assign aesenc_hazard =
        // AESENC 1 instruction ahead: 3-cycle stall needed
        (id_ex_is_aesenc && (id_ex_aesenc_rd != 5'b0) &&
         ((id_ex_aesenc_rd == if_id_rs1) || (id_ex_aesenc_rd == if_id_rs2))) ||
        // AESENC 2 instructions ahead: 2-cycle stall needed
        (ex_mem_is_aesenc && (ex_mem_aesenc_rd != 5'b0) &&
         ((ex_mem_aesenc_rd == if_id_rs1) || (ex_mem_aesenc_rd == if_id_rs2))) ||
        // AESENC 3 instructions ahead: 1-cycle stall needed
        (mem_wb_is_aesenc && (mem_wb_aesenc_rd != 5'b0) &&
         ((mem_wb_aesenc_rd == if_id_rs1) || (mem_wb_aesenc_rd == if_id_rs2)));

    assign stall = load_use || aesenc_hazard;
endmodule
```

A complete implementation also needs extended forwarding: when the AESENC result is finally available (at the end of its 4th execution stage), it must be forwarded to the ALU input of the dependent instruction, which by that point has been stalled (or has an independent instruction inserted between them by the compiler).

---

## Tier 3 — Advanced

### Question A1
**A company is adding a custom RISC-V extension for neural network inference. They want to add a `GEMM_ROW` instruction that computes one row of a matrix-vector multiply. Describe the trade-offs between encoding this as a single custom instruction vs. using the standard V (vector) extension, and outline the verification strategy for the custom approach.**

**Answer:**

**Trade-off analysis:**

**Custom GEMM_ROW instruction:**
```
Hypothetical encoding:
  GEMM_ROW rd, rs1, rs2, imm
  Semantics: rd = dot_product(memory[rs1 : rs1 + imm*4], memory[rs2 : rs2 + imm*4])
             (sum of imm products of 32-bit floats)
  This is effectively a memory-operand instruction with implicit loops.
```

Advantages:
- Extremely compact code: one instruction replaces N multiply-add pairs for a length-N row.
- Custom datapath can be optimised for the exact precision and throughput needed (e.g., INT8 with specific rounding modes for inference).
- Avoids the overhead of vector load/store instructions, `vsetvli`, and the vector register file management.
- Can incorporate the specific weight layout (row-major, column-major, tiled, quantised) directly in hardware.

Disadvantages:
- The instruction has implicit memory accesses and a variable-length loop: the hardware must manage memory transactions internally, complicating the pipeline integration.
- Exception handling is complex: if a memory fault occurs during the middle of the row computation, `mepc` must point to the GEMM_ROW instruction, but partial results may have been written. Is rd updated atomically, or partially?
- Not portable: no other RISC-V processor implements GEMM_ROW.
- Toolchain complexity: the compiler cannot reason about what registers are used by the implicit loop (memory bandwidth, cache behaviour).
- Hardware design complexity: the instruction now embeds a complete DMA-like memory access engine.

**Standard V (vector) extension:**
```
RISC-V V code for one row dot product:
  vsetvli t0, a2, e32, m1, ta, ma   # set vlen for 32-bit elements
  vle32.v v1, (a0); add a0, a0, t0*4 # load row chunk
  vle32.v v2, (a1); add a1, a1, t0*4 # load vector chunk
  vfmacc.vv v0, v1, v2              # accumulate: v0 += v1 * v2
  # repeat until full row processed
  vfmv.f.s ft0, v0                  # extract scalar result
  vfredosum.vs v0, v0, vzero        # reduce accumulator
```

Advantages:
- Fully portable to any RV64GCV processor (any SiFive P-series, Ventana, etc.).
- Toolchain support is in upstream GCC and LLVM.
- Well-understood exception model (vector instructions fault at exact element).
- Extensible: the V extension handles matrices, 1D, 2D, and higher-dimensional data.
- Hardware can be shared with other vector workloads (image processing, cryptography).

Disadvantages:
- More instructions to express the same computation: loop overhead, multiple load/store instructions, vsetvli.
- VLEN is fixed at design time; if the row length is not a multiple of VLEN/SEW, the tail handling adds overhead.
- The V extension is complex to implement correctly (1000+ instructions in the spec).
- For very specific quantised formats (INT4, INT2), the V extension may not have native support until a future sub-extension is ratified.

**Recommendation:** Use the V extension if the product will run general ML workloads or if software portability is valued. Use a custom instruction if the product is a dedicated inference accelerator with a fixed, known weight format and the V extension's overhead is measurable (e.g., 20%+ performance gap in benchmarks).

**Verification strategy for the custom approach:**

```
Level 1 — Instruction semantics test:
  Write a C++ reference model implementing GEMM_ROW exactly.
  Generate random matrix-vector pairs; compare custom instruction output to reference.
  Focus on: rounding modes, NaN/Inf handling, boundary row lengths.

Level 2 — Pipeline integration tests:
  Test cases for the hazard detection extension:
  - GEMM_ROW followed immediately by an instruction that reads rd: verify stall count.
  - GEMM_ROW with a RAW on rs1 (base address from preceding instruction): verify forwarding.
  - Exception during GEMM_ROW (page fault on third cache line accessed): verify mepc=GEMM_ROW PC.

Level 3 — Co-simulation:
  Add GEMM_ROW to Spike as a plugin.
  Run 10,000 random instruction streams (riscv-dv style) including GEMM_ROW.
  Compare RTL output with Spike reference instruction-by-instruction.

Level 4 — Compliance-style tests:
  Write tests modelled after riscv-arch-test format:
  - Each test covers one aspect (correct result, forwarding, exception).
  - Results written to a signature region.
  - Signature compared to expected values.

Level 5 — Application-level test:
  Run a full ML inference benchmark (e.g., MobileNet or a small convolution).
  Compare output activations against a floating-point reference (checking error tolerance).
  Measure CPI improvement vs. scalar or V-extension implementation.
```
