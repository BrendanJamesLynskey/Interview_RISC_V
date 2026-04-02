# CSR Registers

## Prerequisites
- RISC-V privilege modes (M, S, U) and mode transitions
- Basic RISC-V instruction formats (I-type encoding)
- Two's complement arithmetic and bitfield manipulation

---

## Concept Reference

### CSR Address Space and Encoding

RISC-V defines a 12-bit CSR address space, yielding 4096 possible registers. The address encoding carries access-control information directly in the address bits:

```
CSR address bits [11:10] — minimum privilege to access:
  00  =>  Unprivileged / User-mode readable
  01  =>  Supervisor-mode
  10  =>  Hypervisor-mode (H-extension)
  11  =>  Machine-mode only

CSR address bit [9:8] — further privilege sub-encoding:
  The combination of [11:8] determines the exact privilege.

CSR address bits [11:10] = 11 and bits [9:8] = 00..11 => M-mode CSRs
CSR address bits [11:10] = 01 => S-mode CSRs
CSR address bits [11:10] = 00 => U-mode CSRs (performance counters)

Read-only CSRs: bits [11:10] = 11 AND bits [9:8] = 11
  e.g. mvendorid (0xF11), marchid (0xF12), mimpid (0xF13)
```

The hardware checks the current privilege level against bits [11:10] on every CSR instruction. An access at insufficient privilege raises an **illegal instruction exception** (mcause = 2), and the CSR is not touched.

### CSR Instruction Set

All four CSR instructions are encoded as I-type with opcode `1110011` (SYSTEM) and funct3 distinguishing the operation:

```
funct3 = 001  =>  CSRRW   (CSR Read-Write)
funct3 = 010  =>  CSRRS   (CSR Read-Set)
funct3 = 011  =>  CSRRC   (CSR Read-Clear)
funct3 = 101  =>  CSRRWI  (CSR Read-Write Immediate)
funct3 = 110  =>  CSRRSI  (CSR Read-Set Immediate)
funct3 = 111  =>  CSRRCI  (CSR Read-Clear Immediate)
```

Instruction encoding:

```
 31          20  19   15  14  12  11    7   6      0
+--------------+-------+------+--------+---------+
|  csr[11:0]   |  rs1  |funct3|   rd   | 1110011 |
+--------------+-------+------+--------+---------+

For immediate variants (CSRRWI/CSRRSI/CSRRCI):
  rs1 field [19:15] holds a 5-bit unsigned immediate (uimm[4:0])
  instead of a register number.
```

Semantics:

```
CSRRW  rd, csr, rs1   =>  t = CSR[csr]; CSR[csr] = rs1;  rd = t
CSRRS  rd, csr, rs1   =>  t = CSR[csr]; CSR[csr] |= rs1; rd = t
CSRRC  rd, csr, rs1   =>  t = CSR[csr]; CSR[csr] &= ~rs1; rd = t

CSRRWI rd, csr, uimm  =>  t = CSR[csr]; CSR[csr] = uimm; rd = t
CSRRSI rd, csr, uimm  =>  t = CSR[csr]; CSR[csr] |= uimm; rd = t
CSRRCI rd, csr, uimm  =>  t = CSR[csr]; CSR[csr] &= ~uimm; rd = t
```

Important: if `rd = x0`, the read side-effect is suppressed for CSRRS/CSRRC/CSRRSI/CSRRCI. This allows writing a CSR without a wasted register read. If `rs1 = x0` (or `uimm = 0`) for CSRRS/CSRRC, the write is suppressed, making it a pure CSR read.

Canonical pseudo-instructions:

```asm
csrr  rd, csr        # Read CSR:  CSRRS rd, csr, x0
csrw  csr, rs1       # Write CSR: CSRRW x0, csr, rs1
csrs  csr, rs1       # Set bits:  CSRRS x0, csr, rs1
csrc  csr, rs1       # Clear bits: CSRRC x0, csr, rs1
csrwi csr, uimm      # Write immediate: CSRRWI x0, csr, uimm
csrsi csr, uimm      # Set immediate:   CSRRSI x0, csr, uimm
csrci csr, uimm      # Clear immediate: CSRRCI x0, csr, uimm
```

### mstatus — Machine Status Register (0x300)

`mstatus` is the most important M-mode CSR. It tracks and controls the processor's operating state across privilege transitions.

```
RV32 mstatus field layout (selected fields):

Bit  Name    Description
---  ------  --------------------------------------------------
0    WPRI    (reserved, write-preserve read-ignore)
1    SIE     S-mode interrupt enable
2    WPRI
3    MIE     M-mode interrupt enable
4    WPRI
5    SPIE    S-mode previous interrupt enable (saved SIE on trap)
6    UBE     U-mode memory endianness (0=little)
7    MPIE    M-mode previous interrupt enable (saved MIE on trap)
8    SPP     S-mode previous privilege (0=U, 1=S)
9    VS[1]   Vector extension state
10   VS[0]   Vector extension state
11   MPP[0]  M-mode previous privilege (low bit)
12   MPP[1]  M-mode previous privilege (high bit) 00=U,01=S,11=M
13   FS[0]   Floating-point state (Off/Initial/Clean/Dirty)
14   FS[1]   Floating-point state
15   XS[0]   User-mode extension state summary
16   XS[1]   User-mode extension state summary
17   MPRV    Modify PRiVilege (load/store use MPP mode for PMP)
18   SUM     Supervisor User Memory access (allow S-mode access to U pages)
19   MXR     Make eXecutable Readable (allow read of execute-only pages)
20   TVM     Trap Virtual Memory (trap satp/SFENCE.VMA from S-mode)
21   TW      Timeout Wait (trap WFI from S-mode after timeout)
22   TSR     Trap SRET (trap SRET from S-mode)
```

The `FS` and `VS` fields implement **lazy context switching** — the kernel only saves/restores floating-point or vector registers if these fields are non-zero (Dirty), avoiding unnecessary register saves on every context switch.

For RV64, `mstatus` is 64 bits and adds `UXL`/`SXL` fields that control the effective XLEN for U-mode and S-mode code.

### mtvec — Machine Trap Vector Base Address Register (0x305)

`mtvec` holds the trap handler address and the trap vector mode:

```
Bits [XLEN-1:2]  BASE   Handler base address (must be 4-byte aligned)
Bits [1:0]       MODE   0 = Direct, 1 = Vectored, 2-3 = reserved

Direct mode (MODE=0):
  All traps jump to BASE.
  PC = BASE

Vectored mode (MODE=1):
  Synchronous exceptions: PC = BASE
  Asynchronous interrupts: PC = BASE + 4 * cause
  (cause is the interrupt cause number from mcause)
```

Vectored mode eliminates the trap handler's initial branch table for interrupt dispatch, reducing interrupt latency by a few cycles.

Example configuration:

```asm
# Install trap handler at address 0x80001000, direct mode
la   t0, trap_handler    # load address
andi t0, t0, ~3          # clear low 2 bits (ensure 4-byte alignment)
ori  t0, t0, 0           # MODE=0 (direct)
csrw mtvec, t0

# Vectored mode at same address:
ori  t0, t0, 1           # MODE=1 (vectored)
csrw mtvec, t0
```

### mepc — Machine Exception Program Counter (0x341)

`mepc` stores the PC of the instruction that was interrupted or that caused the exception. On `MRET`, the PC is restored to this value.

- For synchronous exceptions: `mepc` = the PC of the faulting instruction.
- For interrupts: `mepc` = the PC of the instruction that was about to execute when the interrupt fired (the instruction that will resume after interrupt handling).

The specification requires `mepc` to be WARL (Write-Any Read-Legal): the hardware may clear the low bits to enforce alignment. On RV32 with no C extension, bit 0 is always 0. On implementations supporting C extension, bit 0 may be 0 or 1 (16-bit instructions are valid).

```asm
# Advance past a faulting instruction (e.g., emulating an unsupported CSR):
csrr  t0, mepc
addi  t0, t0, 4      # skip the 4-byte faulting instruction
csrw  mepc, t0
mret                 # return to instruction after the fault
```

### mcause — Machine Cause Register (0x342)

`mcause` identifies the cause of the most recent trap taken to M-mode.

```
Bit [XLEN-1]  Interrupt  1 = interrupt, 0 = exception
Bits [XLEN-2:0]  Exception/interrupt code

Selected exception codes (Interrupt=0):
  0   Instruction address misaligned
  1   Instruction access fault
  2   Illegal instruction
  3   Breakpoint (EBREAK)
  4   Load address misaligned
  5   Load access fault
  6   Store/AMO address misaligned
  7   Store/AMO access fault
  8   Environment call from U-mode (ecall in U-mode)
  9   Environment call from S-mode (ecall in S-mode)
  11  Environment call from M-mode (ecall in M-mode)
  12  Instruction page fault
  13  Load page fault
  15  Store/AMO page fault

Selected interrupt codes (Interrupt=1):
  1   Supervisor software interrupt
  3   Machine software interrupt
  5   Supervisor timer interrupt
  7   Machine timer interrupt
  9   Supervisor external interrupt
  11  Machine external interrupt
```

Reading mcause in a trap handler:

```asm
trap_handler:
    csrr  t0, mcause
    bltz  t0, is_interrupt        # sign bit set = interrupt
    # handle exception: cause code in t0[XLEN-2:0]
    li    t1, 8                   # ecall from U-mode
    beq   t0, t1, handle_ecall
    j     unexpected_exception
is_interrupt:
    slli  t0, t0, 1               # strip interrupt bit
    srli  t0, t0, 1               # t0 now holds interrupt code
    li    t1, 7                   # machine timer interrupt
    beq   t0, t1, handle_timer
```

### mtval — Machine Trap Value Register (0x343)

`mtval` provides additional context for certain exceptions:

- **Instruction address misaligned / access fault / page fault**: `mtval` = faulting PC.
- **Load/store address misaligned / access fault / page fault**: `mtval` = faulting effective address.
- **Illegal instruction**: `mtval` = the encoding of the illegal instruction (allows software emulation).
- **Breakpoint**: `mtval` = address of the EBREAK instruction.
- **Other exceptions**: `mtval` = 0.

### mip / mie — Interrupt Pending / Interrupt Enable (0x344, 0x304)

These two CSRs control interrupt delivery:

```
mie  (interrupt enable)  — software-controlled mask
mip  (interrupt pending) — hardware sets bits when interrupts arrive

Bit  Name   Description
3    MSIE   M-mode software interrupt enable/pending
7    MTIE   M-mode timer interrupt enable/pending
11   MEIE   M-mode external interrupt enable/pending
1    SSIE   S-mode software interrupt enable/pending
5    STIE   S-mode timer interrupt enable/pending
9    SEIE   S-mode external interrupt enable/pending
```

An interrupt fires when: `mstatus.MIE = 1` AND `mie[bit] = 1` AND `mip[bit] = 1`.

`mip.MTIP` (machine timer interrupt pending) is a read-only bit set by hardware when `mtime >= mtimecmp`. It is cleared by writing a new value to `mtimecmp` (a memory-mapped register, not a CSR).

### mscratch — Machine Scratch Register (0x340)

`mscratch` is a general-purpose register reserved for M-mode trap handler use. By convention it holds a pointer to a per-hart M-mode trap frame — a block of memory where the trap handler saves caller registers.

The canonical use pattern:

```asm
# On trap entry: swap t0 with mscratch (which points to trap frame)
csrrw  t0, mscratch, t0   # t0 = trap frame pointer; mscratch = old t0
sw     t1,  4(t0)          # save t1
sw     t2,  8(t0)          # save t2
# ... save other registers using t0 as the base ...
sw     t0_orig, 0(t0)      # save original t0 (now in mscratch)
# handle trap ...
# On trap return: restore registers, then restore t0
csrrw  t0, mscratch, t0   # restore t0; put trap frame ptr back in mscratch
mret
```

### Supervisor-mode CSR Mirrors

S-mode CSRs at `0x100`–`0x1FF` are either independent registers or restricted views of their M-mode counterparts:

| S-mode CSR | Address | Relationship to M-mode |
|---|---|---|
| `sstatus`  | 0x100 | Restricted view of `mstatus` (only SIE, SPIE, SPP, FS, SUM, MXR, SD) |
| `stvec`    | 0x105 | Independent — S-mode trap handler address |
| `sscratch` | 0x140 | Independent — S-mode scratch register |
| `sepc`     | 0x141 | Independent — S-mode exception PC |
| `scause`   | 0x142 | Independent — S-mode trap cause |
| `stval`    | 0x143 | Independent — S-mode trap value |
| `sip`      | 0x144 | Restricted view of `mip` (only S-mode bits visible) |
| `sie`      | 0x104 | Restricted view of `mie` (only S-mode bits visible) |
| `satp`     | 0x180 | Page table base register (MODE + ASID + PPN) |

`sstatus` is important: S-mode code reading `sstatus` sees only the bits relevant to S-mode. Writing `sstatus.MIE` from S-mode has no effect — the bit is not present in the S-mode view. This enforces the privilege boundary without requiring separate hardware registers for every field.

### satp — Supervisor Address Translation and Protection (0x180)

`satp` enables and configures virtual memory:

```
RV32 satp:
  Bit 31     MODE    0 = Bare (no translation), 1 = Sv32
  Bits 30:22 ASID    9-bit Address Space ID (for TLB tagging)
  Bits 21:0  PPN     22-bit physical page number of root page table

RV64 satp:
  Bits 63:60 MODE    0=Bare, 8=Sv39, 9=Sv48, 10=Sv57
  Bits 59:44 ASID    16-bit Address Space ID
  Bits 43:0  PPN     44-bit physical page number of root page table
```

Writing `satp` to change the address space requires a subsequent `SFENCE.VMA` to flush TLB entries associated with the old ASID.

### Key Read-Only CSRs

```
mvendorid (0xF11)  — JEDEC vendor ID of the core implementer
marchid   (0xF12)  — open-source architecture ID (assigned by RISC-V International)
mimpid    (0xF13)  — implementation version (vendor-defined)
mhartid   (0xF14)  — hardware thread ID; hart 0 is the bootstrap processor
mconfigptr(0xF15)  — pointer to platform configuration structure (Smstateen)

misa      (0x301)  — ISA extensions supported (writable on some implementations
                     to disable extensions at runtime)
```

`mhartid` is essential in multi-core initialisation: all harts start at the same reset vector, and typically only hart 0 (`mhartid = 0`) runs the full boot sequence while others spin on a mailbox.

---

## Tier 1 — Fundamentals

### Question F1
**Name the four CSR instructions and state what each does to the CSR value.**

**Answer:**

| Instruction | Operation on CSR |
|---|---|
| `CSRRW rd, csr, rs1` | Atomically: read old CSR into `rd`, replace CSR with `rs1` |
| `CSRRS rd, csr, rs1` | Atomically: read old CSR into `rd`, set bits in CSR where `rs1` is 1 |
| `CSRRC rd, csr, rs1` | Atomically: read old CSR into `rd`, clear bits in CSR where `rs1` is 1 |
| `CSRRWI/CSRRSI/CSRRCI` | Same as above, but the operand is a 5-bit unsigned immediate instead of `rs1` |

Key side-effect rules:
- `CSRRS`/`CSRRC` with `rs1 = x0`: the CSR is **not written** (read-only access). This is the canonical way to read a CSR without side effects.
- `CSRRW` with `rd = x0`: the old CSR value is **not read** (write-only). Use this when you want to write a CSR without reading, and the read might have side effects (e.g., reading `fflags` clears sticky bits on some implementations).

---

### Question F2
**What is `mcause` and how does a trap handler use it to dispatch to the correct handler?**

**Answer:**

`mcause` (address `0x342`) holds the reason for the most recent trap taken to M-mode. Its top bit (bit XLEN-1) distinguishes interrupts from exceptions:

- Top bit = 1: the trap was an asynchronous interrupt; the remaining bits are the interrupt code.
- Top bit = 0: the trap was a synchronous exception; the remaining bits are the exception code.

A typical trap dispatcher:

```asm
global_trap_handler:
    # Save registers (via mscratch trick) ...

    csrr  t0, mcause
    srli  t1, t0, 31        # isolate top bit (RV32; use 63 for RV64)
    bnez  t1, handle_interrupt

handle_exception:
    li    t1, 2
    beq   t0, t1, handle_illegal_insn
    li    t1, 8
    beq   t0, t1, handle_ecall_u
    li    t1, 9
    beq   t0, t1, handle_ecall_s
    li    t1, 13
    beq   t0, t1, handle_load_page_fault
    li    t1, 15
    beq   t0, t1, handle_store_page_fault
    j     unhandled_exception

handle_interrupt:
    andi  t0, t0, 0x7FFFFFFF  # strip interrupt bit
    li    t1, 7
    beq   t0, t1, handle_timer
    li    t1, 11
    beq   t0, t1, handle_external_irq
    j     unhandled_interrupt
```

**Common mistake:** Treating `mcause` as purely a small integer. The sign bit is not part of the numeric cause code — masking it off is required before comparing interrupt codes numerically.

---

### Question F3
**What does `mepc` contain after an `ecall` instruction? What does the trap handler do to return to the instruction after the `ecall`?**

**Answer:**

After an `ecall`, `mepc` contains the address of the `ecall` instruction itself (the faulting instruction). Unlike some architectures where the saved PC points past the faulting instruction, RISC-V always saves the address of the instruction that caused the exception.

To return to the instruction after the `ecall`, the handler must advance `mepc` by 4 (since `ecall` is always a 32-bit instruction, even on systems with the C extension):

```asm
handle_ecall_u:
    # ... perform system call work using a0-a7 ...

    # Advance past the ecall instruction
    csrr  t0, mepc
    addi  t0, t0, 4
    csrw  mepc, t0

    mret   # or sret if this is the S-mode handler
```

If the handler does not advance `mepc`, `MRET` will return to the `ecall` instruction, causing an infinite trap loop.

---

### Question F4
**What is the purpose of `mscratch`? Write the standard entry/exit sequence for an M-mode trap handler using `mscratch`.**

**Answer:**

`mscratch` is a free-use M-mode register that software is expected to set up before enabling traps. By convention it holds a pointer to a **per-hart trap frame** — a region of M-mode stack or a static structure where the trap handler can save registers without disturbing the interrupted code's register state.

The trap entry problem: the trap handler needs at least one register to work with, but all registers belong to the interrupted code. `CSRRW` solves this atomically:

```asm
# Before enabling interrupts, initialise mscratch:
la    t0, m_trap_frame   # address of 32-register save area
csrw  mscratch, t0

# M-mode trap handler:
trap_entry:
    csrrw  t0, mscratch, t0  # swap: t0 <-> mscratch
                              # now t0 = trap frame ptr, mscratch = old t0
    sw     ra,  0*4(t0)
    sw     sp,  1*4(t0)
    sw     gp,  2*4(t0)
    sw     tp,  3*4(t0)
    # save t1-t6, a0-a7, s0-s11...
    # save original t0, which is now in mscratch:
    csrr   t1, mscratch
    sw     t1,  5*4(t0)      # 5*4 = offset for t0 in frame
    # call C handler with pointer to frame:
    mv     a0, t0
    call   m_trap_c_handler
    # restore:
    lw     ra,  0*4(t0)
    # ... restore all registers except t0 ...
    lw     t1,  5*4(t0)      # original t0
    csrrw  t0, mscratch, t1  # restore t0; put frame ptr back in mscratch
    mret
```

---

### Question F5
**How does the hardware determine whether a CSR access is permitted for the current privilege level?**

**Answer:**

The 12-bit CSR address encodes the minimum access privilege in its top bits. The hardware performs this check on every CSR instruction before reading or writing:

```
CSR address [11:10]:
  00 or 01  =>  accessible from U-mode and above
  10        =>  accessible from S-mode (or H-mode) and above
  11        =>  accessible from M-mode only

Additionally, [11:10] = 11 with [9:8] = 11 => read-only CSR
  Attempt to write a read-only CSR raises illegal instruction exception.
```

If the current privilege level is lower than required by `csr[11:10]`, the instruction raises an **illegal instruction exception** (`mcause = 2`) and the CSR state is not changed. This check happens before any other CSR-specific logic.

Example: S-mode code executing `csrr t0, mstatus` (address `0x300`, bits [11:10] = `11`) raises an illegal instruction exception because S-mode (privilege 01) is below M-mode (privilege 11).

---

## Tier 2 — Intermediate

### Question I1
**Explain the difference between `CSRRS rd, csr, x0` and `CSRRW x0, csr, x0`. When would you use each?**

**Answer:**

`CSRRS rd, csr, x0` — Read CSR, no write:
- Reads the CSR into `rd`.
- Because `rs1 = x0`, the set-bits operation is a no-op (OR with zero).
- The write to the CSR is suppressed entirely, making this a **side-effect-free read**.
- This is the canonical `csrr rd, csr` pseudo-instruction.

`CSRRW x0, csr, x0` — Write zero to CSR, discard read:
- Writes `x0` (zero) to the CSR.
- Because `rd = x0`, the old CSR value is discarded (not read).
- This writes zero to the CSR — it is **not** a no-op.
- Use this only when you deliberately want to zero a CSR (e.g., `csrw fcsr, x0` to clear all floating-point flags and rounding mode).

Practical distinction: to atomically read and zero a CSR (e.g., clearing `fflags` while capturing the old flags value):

```asm
csrrw  t0, fflags, x0   # t0 = old flags; fflags = 0
```

This is different from the no-op read followed by a separate write: the `CSRRW` does both atomically.

---

### Question I2
**Describe the `mstatus.FS` field. How does it enable lazy floating-point context switching in a Linux kernel?**

**Answer:**

`mstatus.FS` (bits 14:13) tracks the state of the floating-point registers:

```
00  Off       FP instructions raise illegal instruction exception
01  Initial   FP registers are in their architectural reset state
10  Clean     FP registers hold values that match the last saved context
11  Dirty     FP registers have been modified since last context save
```

The Linux kernel uses `FS` to implement **lazy FPU context switching**, which avoids saving and restoring 32 floating-point registers (plus `fcsr`) on every context switch:

1. **On context switch out:** the kernel checks `sstatus.FS`. If `FS = Clean` or `FS = Off`, no save is needed — the FP state either matches the saved context or has not been used. If `FS = Dirty`, the kernel saves all FP registers to the outgoing task's `task_struct` and sets `FS = Off` for the incoming task.

2. **On first FP instruction in new context:** With `FS = Off`, the first floating-point instruction raises an illegal instruction exception. The kernel's trap handler loads the FP register state from the incoming task's `task_struct`, sets `FS = Clean`, and re-executes the instruction.

3. **While FP registers are in use:** The hardware sets `FS = Dirty` on any FP instruction that modifies a register. This is the signal to save on the next context switch.

The net effect: tasks that never use floating point incur zero FP save/restore overhead. Tasks that do use FP pay the save/restore cost only on context switches where FP was actually used.

The `VS` field (bits 10:9) provides the same mechanism for the V (vector) extension.

---

### Question I3
**What is the `mstatus.MPRV` bit? Give a practical use case.**

**Answer:**

`MPRV` (Modify PRiVilege, bit 17 of `mstatus`) changes the effective privilege level used for **data** memory accesses (loads and stores) when in M-mode. When `MPRV = 1`, data memory accesses use the privilege level in `mstatus.MPP` instead of M-mode for PMP and page table checks.

`MPRV` does not affect instruction fetches — code always executes at the current (M-mode) privilege.

Practical use case — **kernel `copy_to_user` / `copy_from_user`:**

An S-mode kernel handles a system call that requires reading data from a U-mode virtual address. Normally the kernel cannot directly use U-mode page table mappings. The OpenSBI or the kernel can temporarily set `MPRV = 1` with `MPP = U` in M-mode, then perform the load — the hardware walks the page table as if in U-mode, enforcing U-mode memory protections, before the kernel processes the data.

```asm
# M-mode: temporarily act as U-mode for a load
li    t0, MSTATUS_MPRV    # bit 17
csrs  mstatus, t0         # set MPRV
# mstatus.MPP must already be set to U (00)
lw    a0, 0(a1)           # load from U-mode virtual address a1
csrc  mstatus, t0         # clear MPRV immediately after
```

This is also used by emulation routines that need to simulate the exact memory access behaviour of a lower privilege level, including page fault generation, without a full mode switch.

---

### Question I4
**Explain vectored interrupt mode in `mtvec`. What PC does the processor jump to for a machine timer interrupt in vectored mode?**

**Answer:**

When `mtvec` is configured in vectored mode (bits [1:0] = `01`), the processor computes the trap target address differently for interrupts versus exceptions:

```
Synchronous exceptions (mcause bit[XLEN-1] = 0):
  PC = mtvec[XLEN-1:2] << 2    (BASE, aligned down to 4 bytes)

Asynchronous interrupts (mcause bit[XLEN-1] = 1):
  PC = (mtvec[XLEN-1:2] << 2) + 4 * cause_code
  where cause_code = mcause[XLEN-2:0]
```

For a machine timer interrupt: `mcause = 0x80000007` (interrupt bit set, code = 7).

```
PC = BASE + 4 * 7 = BASE + 28
```

The interrupt vector table in memory must therefore have entries at `BASE + 0`, `BASE + 4`, ..., `BASE + 4*N`. Each entry is typically a 4-byte `JAL x0, handler_N` or a 4-byte jump to the real handler.

Advantage: the hardware does the dispatch directly — no compare/branch chain in a shared trap entry. This saves 5–10 cycles of interrupt latency on a simple in-order core.

Disadvantage: the BASE address must be chosen so that `BASE + 4 * max_cause` does not overlap other code or data. For a platform with up to cause 63, the table is 256 bytes.

---

### Question I5
**What is `misa` and what can software learn from it? Is it always writable?**

**Answer:**

`misa` (Machine ISA, address `0x301`) encodes the supported ISA extensions as a bitmask. Its format:

```
Bits [XLEN-1:XLEN-2]  MXL    Base ISA width: 01=RV32, 10=RV64, 11=RV128
Bits [25:0]           Extensions  Bit 0 = A, Bit 2 = C, Bit 3 = D, bit 4 = E,
                                  Bit 5 = F, Bit 7 = I, Bit 12 = M, Bit 13 = N,
                                  Bit 18 = S, Bit 20 = U, Bit 23 = X (non-standard)
```

Example: `misa = 0x40141105` on RV32IMAC:
```
MXL = 01 (RV32)
Extensions: I (bit 8), M (bit 12), A (bit 0), C (bit 2), S (bit 18), U (bit 20)
```

Writability: the specification allows implementations to make `misa` fully read-only, partially writable, or fully writable (WARL — Write-Any-Read-Legal). A writable `misa` allows software to **disable extensions at runtime**:

- Clearing the `F` bit disables floating-point, causing FP instructions to raise illegal instruction exceptions. Useful for emulating an environment without FP (e.g., for compatibility testing).
- Clearing the `C` bit disables compressed instructions. On some implementations this also disables 2-byte-aligned traps.

Most production implementations make `misa` read-only because dynamically changing the ISA mid-execution has complex implications for the pipeline state.

---

## Tier 3 — Advanced

### Question A1
**A trap handler reads `mcause = 0x00000002` (illegal instruction). It reads `mtval` to get the instruction encoding. Describe how M-mode firmware can use this to emulate an unsupported CSR instruction and why this is architecturally clean.**

**Answer:**

When an illegal instruction exception fires, `mtval` holds the encoding of the faulting instruction. For an unsupported CSR, the hardware does not know the CSR, so it traps to M-mode, which can then decode the instruction and perform the operation in software.

Step-by-step emulation:

```asm
handle_illegal_insn:
    csrr   t0, mtval        # get the faulting instruction encoding
    # Check if it is a CSR instruction: opcode = 1110011 (0x73), funct3 != 0
    andi   t1, t0, 0x7F     # extract opcode
    li     t2, 0x73
    bne    t1, t2, not_csr_insn

    srli   t1, t0, 12       # extract funct3
    andi   t1, t1, 0x7
    beqz   t1, not_csr_insn  # funct3=0 is ECALL/EBREAK, not CSR

    srli   t2, t0, 20       # extract CSR address [31:20]

    # Extract rd [11:7]
    srli   t3, t0, 7
    andi   t3, t3, 0x1F

    # Dispatch based on CSR address in t2
    li     t4, 0xFC0        # example: cycle (U-mode, but emulated)
    beq    t2, t4, emulate_cycle
    # ... other emulated CSRs ...

emulate_cycle:
    # Read the emulated counter value from M-mode storage
    la     t5, m_cycle_shadow
    lw     t6, 0(t5)
    # Write result to the register indexed by t3 (rd field)
    # This requires an indirect register write -- in assembly this needs
    # a jump table or trap frame manipulation:
    la     t4, write_reg_table
    slli   t3, t3, 2
    add    t4, t4, t3
    jr     t4               # jump to register-write stub for rd

    # Advance mepc past the faulting instruction
    csrr   t0, mepc
    addi   t0, t0, 4
    csrw   mepc, t0
    mret
```

Why this is architecturally clean: the RISC-V privilege specification explicitly allows M-mode to emulate any instruction or CSR. The hardware provides `mtval` precisely to support this pattern. Emulation is used in production for:
- Older kernels running on newer hardware with new CSRs the kernel was not compiled to use
- Platform-specific counters that do not have architectural CSR assignments
- Extending the CSR space beyond the 4096 addresses without a hardware change

The pattern is limited by performance: every access to an emulated CSR incurs a full trap, typically 50–200 cycles overhead.

---

### Question A2
**Explain WARL (Write-Any-Read-Legal) CSR semantics. Why does RISC-V specify this for fields like `mtvec.MODE` instead of requiring precise write behaviour?**

**Answer:**

WARL (Write-Any-Read-Legal) means: any value may be written to the field, but the value read back may differ from what was written. The hardware is permitted to ignore unsupported values and substitute the closest legal value.

Example: `mtvec.MODE` field (bits [1:0]):
```
Write 0 (Direct)   =>  read back 0  (universally supported)
Write 1 (Vectored) =>  read back 1 if supported, or 0 if not
Write 2 or 3       =>  read back the closest supported value (typically 0)
```

Software must always **read back after writing** a WARL field to determine what the hardware actually accepted:

```asm
li    t0, 0x80001001      # try vectored mode
csrw  mtvec, t0
csrr  t1, mtvec
andi  t1, t1, 3           # isolate MODE bits
li    t2, 1
bne   t1, t2, vectored_not_supported
# vectored mode accepted
```

Why WARL instead of precise behaviour:
1. **Simplicity of implementation**: An implementation that supports only direct mode does not need any logic to reject or flag vectored-mode writes — it simply ignores the MODE bits and reads back the fixed value.
2. **Forward compatibility**: New mode values can be added in future spec revisions without breaking existing software that reads back and checks.
3. **Reduces spec fragmentation**: Without WARL, every implementation would need to precisely define what happens on every out-of-range write. WARL unifies this into one rule.
4. **Common pattern in hardware**: Hardware registers that have restricted legal values (e.g., alignment-constrained addresses, enumerated modes) naturally behave this way in RTL — writing an unsupported value to a register simply leaves the register unchanged or snaps to a nearby legal value.

Other WARL fields include: `mstatus.MPP` (only 00, 01, 11 are legal), `misa` extension bits (implementation may silently reject clearing certain bits), and PMP configuration registers.
