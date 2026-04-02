# Quiz: ISA Fundamentals

## Instructions

15 multiple-choice questions covering RISC-V instruction formats, RV32I base instructions,
immediate encoding, and the ABI/calling convention. Each question has exactly one correct
answer. Work through all questions before checking the answer key at the end.

Difficulty distribution: Questions 1-5 Fundamentals, Questions 6-11 Intermediate,
Questions 12-15 Advanced.

---

## Questions

### Q1 (Fundamentals)

How many distinct instruction formats are defined in the base RISC-V ISA specification?

- A) 4
- B) 5
- C) 6
- D) 8

---

### Q2 (Fundamentals)

In the RISC-V R-type instruction encoding, which bit field positions hold the destination
register (rd)?

- A) bits [11:7]
- B) bits [19:15]
- C) bits [24:20]
- D) bits [31:27]

---

### Q3 (Fundamentals)

What is the width of all base RV32I instructions?

- A) 16 bits
- B) 24 bits
- C) 32 bits
- D) Variable length

---

### Q4 (Fundamentals)

Which of the following correctly describes the role of register `x0` in RISC-V?

- A) It is the stack pointer and must always hold the current stack address.
- B) It is hardwired to zero; writes are discarded and reads always return 0.
- C) It is the link register used to store return addresses for function calls.
- D) It is a temporary scratch register with no special architectural meaning.

---

### Q5 (Fundamentals)

The RISC-V ABI name for register `x2` is:

- A) `ra`
- B) `gp`
- C) `sp`
- D) `tp`

---

### Q6 (Intermediate)

The `LUI` (Load Upper Immediate) instruction places a 20-bit immediate into bits [31:12]
of the destination register. What value does it place in bits [11:0]?

- A) The lower 12 bits of the PC
- B) The sign extension of bit 19 of the immediate
- C) Zero
- D) The contents of `x0`

---

### Q7 (Intermediate)

Consider the RISC-V S-type instruction format. The 12-bit immediate is split across two
non-contiguous fields. Which fields are they?

- A) `imm[11:5]` in bits [31:25] and `imm[4:0]` in bits [11:7]
- B) `imm[11:5]` in bits [31:25] and `imm[4:0]` in bits [19:15]
- C) `imm[11:0]` packed contiguously in bits [31:20]
- D) `imm[11:5]` in bits [19:13] and `imm[4:0]` in bits [11:7]

---

### Q8 (Intermediate)

What is the maximum signed byte offset that a B-type (branch) instruction can encode
relative to the current PC?

- A) +/- 2 KB (2048 bytes)
- B) +/- 4 KB (4096 bytes)
- C) +/- 1 MB
- D) +/- 2 MB

---

### Q9 (Intermediate)

A programmer writes `ADDI x5, x0, -1`. What value is stored in `x5` after execution on
an RV32I core?

- A) `0x00000001`
- B) `0x7FFFFFFF`
- C) `0xFFFFFFFF`
- D) `0x80000000`

---

### Q10 (Intermediate)

Under the RISC-V calling convention, which registers are guaranteed to be preserved
across a function call (callee-saved)?

- A) `a0`-`a7` (x10-x17)
- B) `t0`-`t6` (x5-x7, x28-x31)
- C) `s0`-`s11` (x8-x9, x18-x27)
- D) `ra` (x1) and `sp` (x2) only

---

### Q11 (Intermediate)

Which instruction would you use to load the full 32-bit address of a symbol `foo` into
register `a0` when the address is not known at compile time?

- A) `LUI a0, foo`
- B) `LI a0, foo` (a pseudo-instruction expanding to `LUI` + `ADDI`)
- C) `AUIPC a0, %pcrel_hi(foo)` followed by `ADDI a0, a0, %pcrel_lo(foo)`
- D) `JAL a0, foo`

---

### Q12 (Advanced)

In the J-type encoding used by `JAL`, the 20-bit immediate is stored in a scrambled
bit order. What is the correct layout from MSB to LSB within the instruction word
(bits [31:12])?

- A) `imm[20|10:1|11|19:12]`
- B) `imm[19:12|11|10:1|20]`
- C) `imm[20|19:12|11|10:1]`
- D) `imm[10:1|11|19:12|20]`

---

### Q13 (Advanced)

A 32-bit I-type instruction word reads `0x00A50513` in hex. Decoding this:

- What is the opcode?
- What operation is performed?
- What is the 12-bit signed immediate?

Which answer is correct?

- A) Opcode `0010011`, `ADDI x10, x10, 10`
- B) Opcode `0000011`, `LW x10, 10(x10)`
- C) Opcode `0010011`, `ADDI x11, x10, 10`
- D) Opcode `0010011`, `SLTI x10, x10, 10`

---

### Q14 (Advanced)

Why does the B-type immediate encode `imm[12:1]` rather than `imm[11:0]`, effectively
making all branch targets multiples of 2?

- A) To allow branches to reach twice as far by treating the offset as a 13-bit value.
- B) Because RISC-V instructions are always 4-byte aligned, so the low bit of any
     instruction address is always 0, giving a free extra bit of range.
- C) Because the C extension allows 16-bit instructions, and branches must be at least
     16-bit aligned, so the lowest bit is implicitly 0.
- D) To reserve bit 0 for a future extension flag.

---

### Q15 (Advanced)

The `FENCE` instruction in RV32I takes `pred` and `succ` operand fields, each 4 bits wide
encoding a combination of I, O, R, W. What do these operands specify?

- A) The predecessor and successor instruction addresses for a memory ordering barrier.
- B) The set of memory accesses (instruction fetch, device I/O, reads, writes) that must
     be ordered: accesses in the `pred` set are guaranteed to be visible before any access
     in the `succ` set to any observer.
- C) The privilege levels between which the fence applies.
- D) Cache hit/miss prediction hints for the hardware prefetcher.

---

## Answer Key

| Q  | Answer | Difficulty    |
|----|--------|---------------|
| 1  | C      | Fundamentals  |
| 2  | A      | Fundamentals  |
| 3  | C      | Fundamentals  |
| 4  | B      | Fundamentals  |
| 5  | C      | Fundamentals  |
| 6  | C      | Intermediate  |
| 7  | A      | Intermediate  |
| 8  | B      | Intermediate  |
| 9  | C      | Intermediate  |
| 10 | C      | Intermediate  |
| 11 | C      | Intermediate  |
| 12 | A      | Advanced      |
| 13 | A      | Advanced      |
| 14 | C      | Advanced      |
| 15 | B      | Advanced      |

---

## Detailed Explanations

### Q1 - Answer: C (6 formats)

The six base formats are R, I, S, B, U, and J. The B-type is sometimes described as a
variant of S-type (they share the same rs1/rs2/funct3 positions) and J-type as a variant
of U-type (they share the same rd/opcode positions), but the spec formally names all six.

- **A (4)** is wrong: that would cover only R, I, S, U, missing B and J.
- **B (5)** is wrong: one of the six is being missed.
- **D (8)** is wrong: there is no such grouping in the base spec.

---

### Q2 - Answer: A (bits [11:7])

In all instruction formats that have a destination register, rd occupies bits [11:7].
This consistent placement means the register file write port address can be read from
the instruction word before full decode is complete.

- **B (bits [19:15])** is the position of rs1.
- **C (bits [24:20])** is the position of rs2.
- **D (bits [31:27])** does not correspond to any standard register field.

---

### Q3 - Answer: C (32 bits)

All base RV32I instructions are exactly 32 bits wide. The two lowest bits of any base
instruction are always `11`. The C (Compressed) extension introduces 16-bit instructions,
but that is a separate optional extension, not part of the base ISA.

- **A (16 bits)** describes C-extension instructions, not base instructions.
- **B (24 bits)** is not a format used in RISC-V.
- **D (Variable length)** is true of the full RISC-V instruction length encoding scheme
  in principle, but the base RV32I set is uniformly 32 bits.

---

### Q4 - Answer: B

`x0` is hardwired to the constant zero. Any write to it is silently ignored, and any
read always returns 0. This is a deliberate architectural choice: it eliminates the need
for a separate "move" instruction (use `ADDI rd, x0, 0`) and allows `NOP` to be
encoded as `ADDI x0, x0, 0`.

- **A** describes `x2` (sp), not `x0`.
- **C** describes `x1` (ra), not `x0`.
- **D** describes the `t` registers (temporaries), not `x0`.

---

### Q5 - Answer: C (sp)

The ABI register aliases: `x0`=zero, `x1`=ra, `x2`=sp, `x3`=gp, `x4`=tp.

- **A (ra)** is `x1`, the return address register.
- **B (gp)** is `x3`, the global pointer.
- **D (tp)** is `x4`, the thread pointer.

---

### Q6 - Answer: C (Zero)

`LUI rd, imm` places the 20-bit immediate in bits [31:12] of `rd` and sets bits [11:0]
to zero. This is designed to be paired with `ADDI rd, rd, lo12` to construct a 32-bit
constant. Because `ADDI` sign-extends its 12-bit immediate, if bit 11 of the low
immediate is set, the programmer must add 1 to the `LUI` value to compensate.

- **A** would make `LUI` PC-relative, which is what `AUIPC` does, not `LUI`.
- **B** would be sign-extension behaviour, which does not apply here.
- **D** is nonsensical; the low bits are not sourced from a register.

---

### Q7 - Answer: A

S-type stores: `imm[11:5]` lives in the `funct7` field at bits [31:25], and `imm[4:0]`
lives in the `rd` field position at bits [11:7]. The split exists so that rs1 and rs2
occupy the same bit positions as they do in R-type instructions, allowing register
file read addresses to be decoded before the instruction type is fully known.

- **B** is wrong because bits [19:15] hold rs1.
- **C** describes I-type, not S-type.
- **D** uses incorrect bit positions.

---

### Q8 - Answer: B (+/- 4 KB)

The B-type immediate encodes a 13-bit signed offset (bits [12:1], with bit 0 implicitly
0). The range is -4096 to +4094 bytes. The maximum forward reach is +4094 bytes and the
maximum backward reach is -4096 bytes, giving an approximately +/- 4 KB range.

- **A (+/- 2 KB)** understates the range by half; 2^12 = 4096 bytes is the one-sided
  magnitude, not 2048.
- **C and D** describe much larger ranges that require U-type or J-type encodings.

---

### Q9 - Answer: C (0xFFFFFFFF)

`ADDI x5, x0, -1` adds the sign-extended 12-bit immediate `-1` (which is `0xFFF` in
12-bit two's complement, sign-extended to `0xFFFFFFFF`) to zero, producing `0xFFFFFFFF`.
This is -1 in 32-bit two's complement, or equivalently, all-bits-set.

- **A (0x00000001)** would result from `ADDI x5, x0, 1`.
- **B (0x7FFFFFFF)** is INT32_MAX and would require a `LUI`+`ADDI` sequence.
- **D (0x80000000)** is INT32_MIN and also requires a multi-instruction sequence.

---

### Q10 - Answer: C (s0-s11)

The RISC-V calling convention designates `s0`-`s11` (`x8`-`x9`, `x18`-`x27`) as
callee-saved. A function that uses any of these registers must save and restore them.
`s0` doubles as the frame pointer `fp`.

- **A (a0-a7)** are argument/return value registers and are caller-saved.
- **B (t0-t6)** are temporaries and are also caller-saved.
- **D** is incomplete: `ra` and `sp` are indeed callee-saved but so are all the `s`
  registers.

---

### Q11 - Answer: C (AUIPC + ADDI)

For position-independent code, the correct pattern is a PC-relative pair:
`AUIPC a0, %pcrel_hi(foo)` loads the upper 20 bits of the PC-relative offset, then
`ADDI a0, a0, %pcrel_lo(foo)` adds the lower 12 bits. The linker fills in the
relocations at link time.

- **A** is incomplete: `LUI a0, foo` only sets the upper 20 bits of an absolute address
  and does not generate a complete 32-bit value.
- **B** (`LI`) is a pseudo-instruction that expands to `LUI`+`ADDI` using an absolute
  address. It is not position-independent and breaks if the binary is relocated.
- **D** (`JAL`) jumps to the address and stores the return address in `a0`; it does not
  load an address as a data value.

---

### Q12 - Answer: A (`imm[20|10:1|11|19:12]`)

The J-type bit scrambling from instruction bits [31:12] is:
- bit 31 = imm[20]
- bits [30:21] = imm[10:1]
- bit 20 = imm[11]
- bits [19:12] = imm[19:12]

This specific scrambling was chosen to maximise overlap with other formats (particularly
B-type) to reduce the hardware cost of the immediate decoder.

- **B** reverses the upper and lower halves.
- **C** would place bits [19:12] adjacent to bit [20], which is not the actual encoding.
- **D** inverts the entire order.

---

### Q13 - Answer: A (`ADDI x10, x10, 10`)

Decoding `0x00A50513`:
- bits [6:0] = `0010011` -> opcode for OP-IMM (integer register-immediate)
- bits [11:7] = `01010` -> rd = x10 (a0)
- bits [14:12] = `000` -> funct3 = 000 -> ADD operation
- bits [19:15] = `01010` -> rs1 = x10 (a0)
- bits [31:20] = `000000001010` -> imm = 10

Result: `ADDI x10, x10, 10` (i.e., `a0 = a0 + 10`).

- **B** would require opcode `0000011` (LOAD group), not `0010011`.
- **C** has the wrong register description in the reasoning but arrives at the same answer
  as A; however option C's description contains an internal inconsistency making it wrong.
- **D** would require funct3 = `010` for SLTI, not `000`.

---

### Q14 - Answer: C

The C extension permits 16-bit instructions, and RISC-V mandates that instruction
fetches be at least 16-bit (2-byte) aligned when C is present. Branch offsets target
instruction addresses, which are always at least 2-byte aligned. Therefore bit 0 of
any target address is always 0, can be inferred implicitly, and the encoding uses
bit 0 of the immediate field to carry `imm[12]` instead, doubling the effective range.

- **A** is partially true (the range does benefit) but the primary reason is alignment,
  not range doubling as a design goal in isolation.
- **B** states instructions are always 4-byte aligned, which is incorrect when the C
  extension is present; 2-byte alignment is the guaranteed minimum.
- **D** invents a future extension flag that does not exist in the spec.

---

### Q15 - Answer: B

`FENCE pred, succ` is a memory ordering instruction. The 4-bit `pred` and `succ` fields
each encode a combination of the letters I (device input), O (device output), R (memory
reads), W (memory writes). The fence ensures that all accesses matching the `pred` bits
are globally visible before any access matching the `succ` bits begins from the
perspective of any other hart or device.

- **A** conflates the operands with instruction addresses; FENCE does not take address
  operands.
- **C** is wrong; privilege level filtering is a function of CSR access rules, not FENCE.
- **D** is wrong; FENCE has no interaction with cache prefetch hints, which are
  implementation-defined and not architecturally visible.
