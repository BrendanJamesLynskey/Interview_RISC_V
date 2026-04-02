# Quiz: Extensions

## Instructions

15 multiple-choice questions covering the RISC-V M, C, F/D, A, B, and V standard
extensions. Each question has exactly one correct answer. Work through all questions
before checking the answer key at the end.

Difficulty distribution: Questions 1-5 Fundamentals, Questions 6-11 Intermediate,
Questions 12-15 Advanced.

---

## Questions

### Q1 (Fundamentals)

What does the letter "M" represent in the RISC-V ISA string `RV32IMC`?

- A) Memory management extension
- B) Integer multiply and divide instructions
- C) Multi-hart (multi-core) support
- D) Misaligned memory access support

---

### Q2 (Fundamentals)

The C extension reduces code size by providing compressed 16-bit encodings for common
instructions. What constraint applies to all C-extension instructions regarding registers?

- A) They can only use even-numbered registers.
- B) Many C instructions (the `CL`/`CS`/`CB`/`CA` format group) are restricted to the
     8 most popular registers `x8`-`x15` for their compact register specifiers.
- C) They can only access the first 8 registers (`x0`-`x7`).
- D) They require register pairs; each operand specifies two consecutive registers.

---

### Q3 (Fundamentals)

Which RISC-V extension provides load-reserved / store-conditional (LR/SC) instructions
and atomic memory operations (AMOs)?

- A) M extension
- B) C extension
- C) A extension
- D) B extension

---

### Q4 (Fundamentals)

The F extension adds single-precision floating-point support. How many floating-point
registers does it define?

- A) 8
- B) 16
- C) 32
- D) 64

---

### Q5 (Fundamentals)

What is the primary purpose of the RISC-V V (Vector) extension?

- A) To provide hardware virtualisation support
- B) To enable SIMD-style data-parallel computation on variable-length vectors
- C) To add volatile memory access semantics
- D) To support variable-length instruction encodings beyond 32 bits

---

### Q6 (Intermediate)

In RV32M, the instruction `MULH rd, rs1, rs2` performs a signed 32x32 multiplication
and returns which part of the 64-bit result?

- A) The lower 32 bits
- B) The upper 32 bits
- C) The lower 16 bits, zero-extended to 32 bits
- D) The full 64-bit result split across rd and rd+1

---

### Q7 (Intermediate)

A C-extension compressed instruction is identified by examining which bits of the
16-bit instruction word?

- A) bits [15:13] (the 3-bit `funct3` field only)
- B) bits [1:0] (the 2-bit `op` field), where a value of `11` indicates a 32-bit
     instruction, and `00`, `01`, or `10` indicate 16-bit C instructions
- C) bit 0 only; `0` means compressed, `1` means base 32-bit
- D) bits [15:14]; a value of `00` signals a C-extension instruction

---

### Q8 (Intermediate)

In the A extension, what is the key semantic difference between an AMO (atomic memory
operation) and an LR/SC (load-reserved / store-conditional) pair?

- A) AMOs operate on 64-bit values only; LR/SC operates on 32-bit values only.
- B) An AMO atomically reads, modifies, and writes a memory location in a single
     indivisible operation. LR/SC implements an optimistic lock: LR reserves the
     address and SC succeeds only if no intervening write has occurred, making it
     suitable for longer critical sections.
- C) LR/SC is mandatory in the A extension; AMOs are optional.
- D) AMOs are only valid in machine mode; LR/SC can be used in any privilege level.

---

### Q9 (Intermediate)

The F and D extensions both use the 32 floating-point registers `f0`-`f31`. What is
the key architectural difference between them in terms of register storage?

- A) F uses 32-bit registers; D uses a separate set of 64-bit registers.
- B) F stores 32-bit single-precision values. D widens the same `f0`-`f31` registers
     to 64 bits; a single-precision value stored via F occupies the lower 32 bits and
     the upper 32 bits are in a NaN-boxing state.
- C) D doubles the number of registers to 64, labelled `f0`-`f63`.
- D) F and D share a unified register file only when both extensions are present;
     otherwise they have separate register files.

---

### Q10 (Intermediate)

The B extension is ratified in three sub-groups. Which of the following is NOT one
of those sub-groups?

- A) Zba (address generation)
- B) Zbb (basic bit manipulation)
- C) Zbc (carry-less multiplication)
- D) Zbd (bit decomposition)

---

### Q11 (Intermediate)

In the V extension, the CSR `vtype` encodes which information about the current vector
configuration?

- A) The total number of vector registers available
- B) The selected element width (SEW) and the vector length multiplier (LMUL)
- C) The base address of the vector register file in memory
- D) The privilege level required to execute vector instructions

---

### Q12 (Advanced)

An RV32M core executes `DIV rd, rs1, rs2` where `rs1 = 0x80000000` (INT32_MIN) and
`rs2 = 0xFFFFFFFF` (-1). What value is written to `rd`?

- A) `0x7FFFFFFF` (INT32_MAX, the mathematically correct result clipped to 32 bits)
- B) `0x80000000` (INT32_MIN, the overflow result as specified by the ISA)
- C) `0x00000001`
- D) The result is architecturally undefined.

---

### Q13 (Advanced)

Consider a sequence using the A extension on an RV64A core:

```
LR.D  t0, (a0)       # load-reserve doubleword at address a0
ADDI  t0, t0, 1
SC.D  t1, t0, (a0)   # store-conditional: t1 = 0 on success, 1 on failure
```

Under what circumstances will `SC.D` fail (write 1 to `t1`)?

- A) Only if another hart has written to address `a0` since the `LR.D`.
- B) If any exception or interrupt has been taken since the `LR.D`, if the address
     `a0` has been written by any hart, or if the reservation has been invalidated
     by any implementation-permitted cause (including cache eviction on some
     implementations).
- C) Only if a context switch has occurred since the `LR.D`.
- D) Never; SC.D is guaranteed to succeed on single-hart systems.

---

### Q14 (Advanced)

The V extension uses a "vector length agnostic" programming model. The instruction
`VSETVLI t0, a0, e32, m2, ta, ma` configures the vector unit. What does the `m2`
parameter specify, and what is its effect on the effective number of elements per
vector operation?

- A) It selects LMUL=2, meaning each logical vector register group spans 2 physical
     vector registers, doubling the number of elements processed per instruction
     compared to LMUL=1.
- B) It selects LMUL=1/2, halving the number of elements per vector operation.
- C) It means the vector unit uses 2-wide SIMD lanes internally, which is
     implementation-defined and not visible to software.
- D) It reserves 2 vector registers as mask registers, reducing the usable register
     count to 30.

---

### Q15 (Advanced)

The C extension includes the pseudo-instruction `C.NOP`. What does it expand to, and
why is this encoding chosen?

- A) It expands to `C.ADDI x0, 0`. The encoding `0x0001` is chosen because writes to
     `x0` are discarded, making the operation definitionally a no-op, and `imm=0` is
     the only legal non-hint value.
- B) It expands to `C.MV x0, x0`. The encoding is chosen to match the all-zeros
     16-bit pattern, which hardware can detect cheaply.
- C) It expands to `C.ADDI x1, 0`, using `ra` because it is always present.
- D) There is no `C.NOP`; the C extension requires using the 32-bit `NOP` even when C
     is enabled.

---

## Answer Key

| Q  | Answer | Difficulty    |
|----|--------|---------------|
| 1  | B      | Fundamentals  |
| 2  | B      | Fundamentals  |
| 3  | C      | Fundamentals  |
| 4  | C      | Fundamentals  |
| 5  | B      | Fundamentals  |
| 6  | B      | Intermediate  |
| 7  | B      | Intermediate  |
| 8  | B      | Intermediate  |
| 9  | B      | Intermediate  |
| 10 | D      | Intermediate  |
| 11 | B      | Intermediate  |
| 12 | B      | Advanced      |
| 13 | B      | Advanced      |
| 14 | A      | Advanced      |
| 15 | A      | Advanced      |

---

## Detailed Explanations

### Q1 - Answer: B (Integer multiply and divide)

The M extension adds `MUL`, `MULH`, `MULHSU`, `MULHU`, `DIV`, `DIVU`, `REM`, `REMU`
instructions. The base RV32I deliberately excludes these to keep the minimal integer
core as simple as possible.

- **A** is wrong: memory management is part of the privilege spec (Sv32/Sv39), not an
  ISA extension letter.
- **C** is wrong: multi-hart support is an implementation property, not an extension.
- **D** is wrong: misaligned access support is implementation-defined; there is no
  dedicated ISA letter for it.

---

### Q2 - Answer: B

The C extension defines several instruction formats. Many of them (CL, CS, CA, CB) use
3-bit compact register specifiers that can only address 8 registers. Those 8 registers
are `x8`-`x15` (which map to ABI names `s0`, `s1`, `a0`-`a5`). These were chosen as
the most frequently used registers in typical code, maximising compression effectiveness.

The CI and CR formats have access to all 32 registers via a 5-bit specifier.

- **A** (even-numbered only) is not an architectural constraint.
- **C** (`x0`-`x7` only) has the wrong register range; the restricted set is `x8`-`x15`.
- **D** (register pairs) is not a feature of the C extension.

---

### Q3 - Answer: C (A extension)

The A (Atomics) extension provides LR.W/LR.D, SC.W/SC.D, and the AMO family
(AMOSWAP, AMOADD, AMOXOR, AMOAND, AMOOR, AMOMIN, AMOMAX, AMOMINU, AMOMAXU).

- **A (M)** provides multiply/divide only.
- **B (C)** provides compressed 16-bit encodings.
- **D (B)** provides bit manipulation operations.

---

### Q4 - Answer: C (32 registers)

The F extension adds 32 floating-point registers `f0`-`f31`, each 32 bits wide. This
is the same count as the integer register file. The D extension widens these same
registers to 64 bits rather than adding new ones.

---

### Q5 - Answer: B

The V extension provides variable-length vector operations that process multiple data
elements in parallel. The key innovation over traditional SIMD is that vector length is
configurable at runtime via `vsetvl`, allowing the same code to run efficiently on
implementations with different physical vector widths.

- **A** is wrong: hardware virtualisation is handled by the H (Hypervisor) extension.
- **C** is wrong: volatile semantics are not part of the ISA; that is a software/language concern.
- **D** is wrong: variable-length instruction encodings are handled by the base ISA
  length encoding scheme, not the V extension.

---

### Q6 - Answer: B (upper 32 bits)

`MULH rd, rs1, rs2` performs a signed 32-bit by signed 32-bit multiplication producing
a 64-bit result, then writes the upper 32 bits to `rd`. This allows software to perform
full 64-bit multiplication on an RV32M core using a `MUL`/`MULH` pair.

- **A** describes `MUL`, which returns the lower 32 bits.
- **C** is not a real operation.
- **D** would require a 64-bit destination; RISC-V does not have implicit register pairs.

---

### Q7 - Answer: B (bits [1:0])

The RISC-V instruction length encoding uses the lowest bits of the instruction word.
For 16-bit C instructions, bits [1:0] are `00`, `01`, or `10`. For standard 32-bit
instructions, bits [1:0] are always `11`. This allows hardware to determine instruction
length before completing the fetch.

- **A** is wrong: `funct3` alone does not distinguish 16-bit from 32-bit instructions.
- **C** is wrong: a single bit is insufficient to encode the three valid C quadrants.
- **D** is wrong: bits [15:14] are part of the `funct3`/opcode fields in C instructions,
  not a length indicator.

---

### Q8 - Answer: B

AMOs (e.g., `AMOADD.W`) are true hardware-atomic read-modify-write operations: the
memory bus transaction is indivisible. LR/SC works differently: `LR` reads and reserves
an address, and `SC` succeeds only if the reservation is still valid, allowing the
processor to retry on failure. LR/SC is preferred for operations that require reading
a value and conditionally writing based on it (compare-and-swap, etc.).

- **A** is wrong: both AMOs and LR/SC have 32-bit (.W) and 64-bit (.D) variants.
- **C** is wrong: AMOs are equally mandatory within the A extension.
- **D** is wrong: both AMOs and LR/SC are available in all privilege modes (subject to
  PMP and PMA constraints).

---

### Q9 - Answer: B (NaN-boxing)

When F and D coexist, the 64-bit `f` registers store 32-bit single-precision values in
the lower 32 bits with the upper 32 bits set to all-ones (the NaN-boxing convention).
Any operation that reads a 32-bit value from a 64-bit register checks that the upper 32
bits are all-ones; if not, the value is treated as a canonical NaN.

- **A** is wrong: there is only one set of 32 `f` registers shared by F and D.
- **C** is wrong: D does not add extra registers.
- **D** is wrong: the same register file is used whether F, D, or both are present.

---

### Q10 - Answer: D (Zbd does not exist)

The ratified B extension sub-groups are: Zba (address generation: `SH1ADD`, `SH2ADD`,
`SH3ADD`, `ADD.UW`), Zbb (basic bit manipulation: `CLZ`, `CTZ`, `CPOP`, `ANDN`,
`ORN`, `XNOR`, rotates, byte reversal, sign extension), and Zbc (carry-less multiply:
`CLMUL`, `CLMULH`, `CLMULR`). There is no "Zbd" sub-group.

---

### Q11 - Answer: B (SEW and LMUL)

`vtype` encodes the selected element width (SEW: e8, e16, e32, e64) and the vector
length multiplier (LMUL: m1, m2, m4, m8, mf2, mf4, mf8), along with tail and mask
agnostic policies. Together with `vl` (vector length), these determine the exact
behaviour of every vector instruction.

- **A** is wrong: the number of vector registers is a fixed implementation parameter,
  not stored in a CSR.
- **C** is wrong: vector registers are not memory-mapped.
- **D** is wrong: privilege requirements for vector instructions are not encoded in `vtype`.

---

### Q12 - Answer: B (0x80000000)

The RISC-V spec explicitly defines the overflow case for signed division: when the
dividend is INT32_MIN (`0x80000000`) and the divisor is -1, the mathematical result
(INT32_MAX + 1) overflows 32 bits. The spec mandates that the result is the dividend
itself (INT32_MIN = `0x80000000`), matching the behaviour of x86 and most hardware.
The remainder is defined as 0 in this case.

- **A** is wrong: the spec does not saturate to INT32_MAX; it returns the dividend.
- **C** is wrong.
- **D** is wrong: the behaviour is fully defined; RISC-V has no integer undefined
  behaviour for division overflow.

---

### Q13 - Answer: B

The RISC-V specification lists several conditions under which SC may fail, not just
a write from another hart. These include: taking any trap between LR and SC, a write
from any hart to the reserved granule, and any implementation-defined condition. An SC
is only guaranteed to succeed if the reservation granule has not been written and no
trap was taken. Single-hart systems can still fail an SC due to timer interrupts or
other traps taken between LR and SC.

- **A** is too narrow: traps and implementation-specific causes also cause failure.
- **C** is too narrow: any trap (not just context switches) can cause failure.
- **D** is wrong: SC on single-hart systems can still fail due to interrupt handling.

---

### Q14 - Answer: A (LMUL=2 doubles elements per operation)

`m2` sets LMUL (vector length multiplier) to 2. Each logical vector register `v0`, `v2`,
`v4`, ... is backed by a group of 2 physical vector registers. With LMUL=2 and SEW=32,
a core with VLEN=128 bits would process 8 elements per instruction (2 x 4 elements)
instead of 4 at LMUL=1. This trades register file capacity (you have 16 logical
registers instead of 32) for increased throughput per instruction on wide vector units.

- **B** describes `mf2` (fractional LMUL), not `m2`.
- **C** is wrong: LMUL is an architectural concept, not an implementation hint.
- **D** is wrong: mask registers are selected by operand, not by LMUL.

---

### Q15 - Answer: A

`C.NOP` is defined as `C.ADDI x0, 0` with encoding `0x0001`. Writing to `x0` is always
discarded by the architecture, and adding zero performs no computation, making the
combined effect a true no-op. The encoding `0x0001` corresponds to `C.ADDI` with
rd=x0 and imm=0.

- **B** is wrong: `C.MV x0, x0` is not a valid encoding in the C extension because
  `C.MV` with rd=x0 is reserved.
- **C** is wrong: `x1` (ra) has no special status that would make it the natural choice
  for a no-op.
- **D** is wrong: `C.NOP` is explicitly defined in the C extension specification.
