# Trap Handling

## Prerequisites
- RISC-V privilege modes (M, S, U) and the purpose of each
- CSR registers: mtvec, mepc, mcause, mtval, mstatus, medeleg, mideleg
- Basic understanding of exceptions and interrupts in computer architecture

---

## Concept Reference

### Traps: Exceptions and Interrupts

RISC-V uses the term **trap** to mean any event that causes the processor to transfer control to a trap handler. Traps are divided into two categories:

```
Exceptions (synchronous):
  Caused by the execution of a specific instruction.
  mepc = address of the causing instruction.
  Deterministic: same instruction sequence always produces the same exception.

  Examples:
    - Illegal instruction (faulting instruction encoding in mtval)
    - Environment call (ecall)
    - Breakpoint (ebreak)
    - Address misaligned (faulting address in mtval)
    - Page fault (faulting address in mtval)
    - Access fault (hardware bus error, faulting address in mtval)

Interrupts (asynchronous):
  Caused by an external event, not by a specific instruction.
  mepc = address of the instruction that was about to execute.
  Non-deterministic: can arrive at any instruction boundary.

  Types:
    - Software interrupt (written to memory-mapped registers)
    - Timer interrupt (mtime >= mtimecmp)
    - External interrupt (PLIC asserts an interrupt line)
```

The distinction between exception and interrupt is visible in `mcause` bit [XLEN-1]: 1 for interrupt, 0 for exception.

### Trap Entry Sequence

When a trap is taken to **M-mode**, the hardware performs the following atomically (no software involvement):

```
1. mcause  <- trap cause (interrupt bit | cause code)
2. mepc    <- PC of interrupted/faulting instruction
3. mtval   <- exception-specific value (faulting address, insn encoding, or 0)
4. mstatus.MPIE <- mstatus.MIE   (save current interrupt enable state)
5. mstatus.MIE  <- 0             (disable M-mode interrupts)
6. mstatus.MPP  <- current privilege level  (save current mode)
7. current privilege <- M-mode
8. PC <- mtvec (direct mode) or mtvec + 4*cause (vectored mode, interrupts only)
```

When a trap is taken to **S-mode** (because it was delegated — see below):

```
1. scause  <- trap cause
2. sepc    <- PC
3. stval   <- exception-specific value
4. sstatus.SPIE <- sstatus.SIE
5. sstatus.SIE  <- 0
6. sstatus.SPP  <- current privilege (0=U, 1=S)
7. current privilege <- S-mode
8. PC <- stvec (direct or vectored)
```

Note: M-mode CSRs (`mepc`, `mcause`, `mstatus`) are **never written** when a trap is taken to S-mode. The hardware writes only the S-mode trap CSRs.

### Trap Return: MRET and SRET

`MRET` (Machine Return) and `SRET` (Supervisor Return) are privileged instructions that return from a trap handler. They atomically reverse the trap entry sequence:

**MRET:**
```
1. PC              <- mepc
2. current privilege <- mstatus.MPP
3. mstatus.MIE    <- mstatus.MPIE    (restore interrupt enable)
4. mstatus.MPIE   <- 1               (set to 1, per spec)
5. mstatus.MPP    <- U               (reset to least privilege)
```

**SRET:**
```
1. PC              <- sepc
2. current privilege <- mstatus.SPP  (0=U, 1=S)
3. sstatus.SIE    <- sstatus.SPIE
4. sstatus.SPIE   <- 1
5. sstatus.SPP    <- U               (reset to 0)
```

`SRET` raises an illegal instruction exception if executed in U-mode. It raises an illegal instruction exception in S-mode if `mstatus.TSR = 1` (Trap SRET, used by hypervisors).

`MRET` executed in U-mode raises an illegal instruction exception.

### Trap Delegation: medeleg and mideleg

By default, all traps are taken to M-mode regardless of the current privilege level. Delegation registers allow M-mode to redirect traps to S-mode, which is essential for running a full OS kernel in S-mode.

```
medeleg (Machine Exception Delegation, 0x302):
  Bit N = 1 means exception cause N is delegated to S-mode.
  Only takes effect when the trap originates in S-mode or U-mode.
  A trap from M-mode always goes to M-mode regardless of medeleg.

mideleg (Machine Interrupt Delegation, 0x303):
  Bit N = 1 means interrupt cause N is delegated to S-mode.
```

When a trap is delegated and the current privilege is S-mode or U-mode:
- The trap is taken to S-mode (not M-mode)
- S-mode trap CSRs are written (sepc, scause, stval, sstatus)
- M-mode trap CSRs are NOT written

Typical delegation configuration for a Linux system (performed by OpenSBI):

```asm
# Delegate most exceptions to S-mode
li    t0, 0x0000B1FF   # bits: misaligned fetch(0), illegal insn(2),
                        # breakpoint(3), misaligned load(4), load fault(5),
                        # misaligned store(6), store fault(7),
                        # ecall from U-mode(8), ecall from S-mode(9),
                        # instruction page fault(12), load page fault(13),
                        # store page fault(15)
csrw  medeleg, t0

# Delegate S-mode interrupts to S-mode
li    t0, 0x00000222   # SSIE(1), STIE(5), SEIE(9)
csrw  mideleg, t0
```

Crucially, **ecall from M-mode (cause 11) is never delegated** — M-mode handles its own ecalls. And once S-mode has delegated traps, it can further delegate to U-mode via `sedeleg`/`sideleg` (if the N extension is implemented, rarely used).

### Priority of Simultaneous Traps

If multiple exceptions occur simultaneously (impossible in normal sequential code but possible in some microarchitectural edge cases), or if multiple interrupts are pending, priority rules apply:

**Exception priority** (higher = taken first):
```
1. Instruction address breakpoint
2. Instruction page fault / instruction access fault
3. Illegal instruction
4. Instruction address misaligned
5. ECALL / EBREAK
6. Load/store address breakpoint
7. Load/store misaligned / access fault / page fault
```

**Interrupt priority within M-mode:**
```
MEI (machine external) > MSI (machine software) > MTI (machine timer)
> SEI > SSI > STI
```

Interrupts take priority over exceptions only at instruction boundaries — a pending interrupt cannot preempt an instruction mid-execution.

### Nested Traps and Re-entrancy

When a trap handler executes, `mstatus.MIE` is cleared, disabling further M-mode interrupts. This prevents a second interrupt from overwriting `mepc` and `mcause` before the handler saves them.

To support **nested interrupts** (higher-priority interrupts preempting lower-priority handlers), the handler must:

1. Save `mepc` and `mstatus` to stack.
2. Re-enable M-mode interrupts (`csrsi mstatus, MSTATUS_MIE`).
3. Handle the interrupt.
4. Disable interrupts before returning.
5. Restore `mepc` and `mstatus` from stack.
6. Execute `MRET`.

```asm
nested_trap_handler:
    # Phase 1: Save context atomically (MIE still clear)
    csrrw  t0, mscratch, t0
    sw     ra, FRAME_RA(t0)
    sw     a0, FRAME_A0(t0)
    csrr   a0, mepc
    sw     a0, FRAME_EPC(t0)
    csrr   a0, mstatus
    sw     a0, FRAME_STATUS(t0)
    mv     a0, t0          # pass frame pointer to C handler

    # Phase 2: Re-enable interrupts for nesting
    csrsi  mstatus, 8      # set MIE (bit 3)

    call   c_trap_handler  # may be interrupted here

    # Phase 3: Restore (disable interrupts first)
    csrci  mstatus, 8      # clear MIE
    lw     t1, FRAME_STATUS(a0)
    csrw   mstatus, t1
    lw     t1, FRAME_EPC(a0)
    csrw   mepc, t1
    lw     ra, FRAME_RA(a0)
    # restore a0 last since it holds the frame pointer
    lw     a0, FRAME_A0(a0)
    csrrw  t0, mscratch, t0
    mret
```

### Interrupt Sources: Software, Timer, External

**Software interrupts** are used for inter-processor interrupts (IPI) in multi-core systems. A hart triggers a software interrupt on another hart by writing to the CLINT (Core Local Interruptor) `msip` register. `mip.MSIP` (bit 3) is set, and if `mie.MSIE` is set and `mstatus.MIE` is set, the target hart takes a machine software interrupt.

**Timer interrupt** is generated when the memory-mapped `mtime` register (a free-running counter) reaches or exceeds `mtimecmp`. `mip.MTIP` (bit 7) is set by hardware and cleared by writing a new value to `mtimecmp`. The timer interrupt is the primary mechanism for preemptive scheduling.

```asm
# Set timer interrupt for 10ms from now (assuming 10 MHz mtime clock)
# mtime and mtimecmp are memory-mapped at platform-defined addresses
li    t0, MTIME_BASE
ld    t1, 0(t0)         # read current mtime (64-bit on both RV32 and RV64)
li    t2, 100000        # 10ms * 10MHz = 100000 ticks
add   t1, t1, t2
la    t0, MTIMECMP_BASE
sd    t1, 0(t0)         # write mtimecmp; this also clears MTIP momentarily

# On RV32, must use two 32-bit writes to avoid spurious interrupt:
li    t3, 0xFFFFFFFF
sw    t3, 4(t0)         # set high word to max first
sw    t1, 0(t0)         # write low word
srli  t2, t1, 32        # extract high word
sw    t2, 4(t0)         # write correct high word
```

**External interrupts** are routed through the PLIC (Platform-Level Interrupt Controller), which arbitrates between many external interrupt sources and presents the highest-priority pending interrupt to each hart. The hart reads the PLIC's claim register to acknowledge the interrupt, handles it, then writes the completion register.

---

## Tier 1 — Fundamentals

### Question F1
**What is the difference between an exception and an interrupt in RISC-V? How does `mcause` distinguish them?**

**Answer:**

An **exception** is synchronous — it is caused by a specific instruction and occurs at a precise point in the instruction stream. Examples: illegal instruction, page fault, ecall.

An **interrupt** is asynchronous — it is caused by an external event that is independent of the currently executing instruction. Examples: timer expiry, external peripheral signal, IPI from another hart.

`mcause` uses its most-significant bit (bit XLEN-1) as a flag:

```
mcause[XLEN-1] = 0  =>  exception; remaining bits = exception code
mcause[XLEN-1] = 1  =>  interrupt; remaining bits = interrupt code

Example values (RV32):
  0x00000002  =>  Illegal instruction (exception, code 2)
  0x00000008  =>  Ecall from U-mode (exception, code 8)
  0x80000007  =>  Machine timer interrupt (interrupt, code 7)
  0x8000000B  =>  Machine external interrupt (interrupt, code 11)
```

**Common mistake:** On RV32, naively comparing `mcause` to 7 to check for a timer interrupt will fail because the value is `0x80000007`, which is negative when treated as a signed integer. The interrupt bit must be stripped or the comparison must use an unsigned check with the full value.

---

### Question F2
**What value does `mepc` hold after a page fault? After a timer interrupt?**

**Answer:**

**After a page fault (exception):**
`mepc` holds the address of the instruction that caused the fault. For a load/store page fault, this is the load or store instruction itself. For an instruction page fault (fetch of a non-mapped page), it is the address that was being fetched.

The trap handler can read `mtval` to get the faulting virtual address that caused the page fault — this is distinct from `mepc`.

**After a timer interrupt (interrupt):**
`mepc` holds the address of the instruction that was about to execute when the interrupt was taken. After the interrupt handler runs, `MRET` will return to this instruction, which will then execute normally.

The interrupt handler does NOT increment `mepc` — the interrupted instruction was not the cause of the trap and must be re-executed.

**Summary:**
```
Exception -> mepc = address of the faulting instruction (re-execute or skip)
Interrupt -> mepc = address of next instruction to execute (always resume here)
```

---

### Question F3
**A RISC-V firmware engineer writes a trap handler but forgets to set `mtvec` before enabling interrupts. What happens when the first interrupt fires?**

**Answer:**

`mtvec` has an implementation-defined reset value — typically 0x00000000 or another platform-specific address. If `mtvec` is never initialised and an interrupt fires, the processor jumps to the reset value address.

Possible outcomes:
- If address 0 contains valid instruction memory (e.g., the reset vector on some MCUs), execution jumps there and the system may appear to reset.
- If address 0 is unmapped or is data memory, the instruction fetch generates an **instruction access fault** exception. This exception is also taken to M-mode, jumping to `mtvec` — which is still 0 — creating an infinite fault loop.
- On systems with PMP blocking address 0, the access fault fires, and the recursive trap loop begins. The system hangs.

Best practice: always set `mtvec` before `mstatus.MIE` is set. The boot sequence should be:

```asm
la   t0, trap_handler
csrw mtvec, t0           # set trap handler first
csrsi mstatus, 8         # only then enable interrupts
```

---

### Question F4
**What does `medeleg` do, and why is it important for running Linux on RISC-V?**

**Answer:**

`medeleg` (Machine Exception Delegation, CSR `0x302`) is a bitmask that allows M-mode firmware to redirect exception handling from M-mode to S-mode. Bit N = 1 means exception cause N goes to S-mode (using `sepc`, `scause`, `stvec`, etc.) when the trap originates in S-mode or U-mode.

Without delegation, every system call, page fault, and illegal instruction executed by a Linux user process would trap to M-mode firmware first, which would then re-enter S-mode to deliver the exception to the kernel. This adds at minimum two privilege transitions per trap (U->M->S->U), dramatically increasing system call and page fault overhead.

With `medeleg` configured:
- `ecall` from U-mode (cause 8) is delegated to S-mode: the trap goes directly to the Linux kernel's system call handler.
- Page faults (causes 12, 13, 15) are delegated: the kernel's page fault handler runs without M-mode involvement.
- Breakpoints (cause 3) are delegated: `gdb` debugging works via the kernel's ptrace interface.

M-mode (OpenSBI) retains handling of:
- `ecall` from S-mode (cause 9): SBI calls from the kernel to M-mode firmware.
- M-mode ecall (cause 11): never delegated.
- Machine check/access faults that require firmware-level intervention.

---

## Tier 2 — Intermediate

### Question I1
**Trace through the complete hardware and software sequence when a U-mode process executes `ecall` on a Linux RISC-V system with standard OpenSBI delegation.**

**Answer:**

Precondition: OpenSBI has set `medeleg[8] = 1` (delegate ecall from U-mode to S-mode).

Step 1 — U-mode `ecall` instruction executes:
```
Hardware:
  scause  = 8 (ecall from U-mode)
  sepc    = address of the ecall instruction
  stval   = 0 (no relevant address for ecall)
  sstatus.SPIE = sstatus.SIE (save SIE)
  sstatus.SIE  = 0 (disable S-mode interrupts)
  sstatus.SPP  = 0 (previous mode was U)
  privilege    = S-mode
  PC           = stvec (kernel's trap entry)
```

Step 2 — Kernel trap entry (`arch/riscv/kernel/entry.S`):
```
Software (S-mode):
  Save all 32 registers + sepc + sstatus to the kernel thread's pt_regs.
  Call handle_exception().
  Dispatch on scause: cause 8 = system call.
```

Step 3 — System call dispatch:
```
Software (S-mode):
  Read syscall number from a7 register (saved in pt_regs).
  Look up sys_call_table[a7].
  Execute the system call function (e.g., sys_write).
  Return value placed in a0 in pt_regs.
```

Step 4 — Return preparation:
```
Software (S-mode):
  Advance sepc by 4 (skip the ecall instruction).
  Restore all registers from pt_regs.
```

Step 5 — `SRET` execution:
```
Hardware:
  PC           = sepc (instruction after ecall)
  privilege    = sstatus.SPP = 0 = U-mode
  sstatus.SIE  = sstatus.SPIE (restore interrupt enable)
  sstatus.SPP  = 0 (reset)
```

Total mode transitions: U -> S -> U. No M-mode involvement for the system call itself.

---

### Question I2
**Explain the `mstatus.SIE` and `mstatus.MIE` interaction. Can S-mode interrupts fire while M-mode is executing with `MIE=1`?**

**Answer:**

`MIE` and `SIE` are separate interrupt enable bits that apply to their respective privilege levels. The interaction follows these rules:

1. **While in M-mode:** Only `mstatus.MIE` controls whether interrupts are taken to M-mode. `SIE` is irrelevant — S-mode interrupts that are pending (SSIP, STIP, SEIP in `mip`) and enabled in `mie` will be taken to M-mode if `MIE=1` and those bits are not delegated. If the interrupt **is** delegated (`mideleg[bit] = 1`), it can only be taken when the privilege level is S-mode or lower, not while in M-mode.

2. **While in S-mode:** `sstatus.SIE` (which is the S-mode view of `mstatus.SIE`) controls whether S-mode interrupts are taken. M-mode interrupts (MTIP, MEIP, MSIP) are always enabled regardless of `SIE` — they will preempt S-mode and go to M-mode if `mie[bit]=1`.

3. **Delegated interrupt in M-mode:** If `mideleg[5] = 1` (timer interrupt delegated to S) and M-mode is running with `MIE=1`, the `STIP` bit does NOT cause an interrupt to M-mode — delegated interrupts are only delivered when the effective privilege is S-mode or lower.

Practical implication: M-mode firmware can safely run with its own interrupts enabled (`MIE=1`) without worrying about S-mode timer interrupts interrupting it — those are pending but will wait until control returns to S-mode.

---

### Question I3
**What is the PLIC and how does the external interrupt flow work from interrupt source to M-mode handler?**

**Answer:**

The **PLIC (Platform-Level Interrupt Controller)** is a standardised memory-mapped interrupt controller defined in the RISC-V Platform Specification. It aggregates multiple interrupt sources and presents the highest-priority pending interrupt to each hart's M-mode and S-mode external interrupt lines.

Full flow for an external interrupt (e.g., a UART RX interrupt):

```
1. UART hardware asserts its interrupt line to the PLIC.

2. PLIC:
   - Sets the interrupt source pending bit.
   - If the source priority > threshold for hart 0 M-mode context,
     PLIC asserts the MEI (Machine External Interrupt) line to hart 0.

3. Hart 0 hardware:
   - mip.MEIP is set by the PLIC assertion.
   - If mstatus.MIE=1 and mie.MEIE=1:
     - Trap to M-mode: mcause = 0x8000000B (MEI)
     - mepc = interrupted PC
     - PC = mtvec

4. M-mode trap handler:
   - Reads the PLIC claim register at PLIC_CLAIM_BASE
     (returns the highest-priority interrupt ID, e.g., UART=4)
   - Dispatches to uart_handler()
   - uart_handler() reads UART data register, clears UART interrupt flag.
   - Writes UART interrupt ID back to PLIC completion register
     (this clears mip.MEIP if no other interrupts are pending)
   - mret

5. If S-mode PLIC context is used (interrupt delegated via mideleg):
   Steps 3-4 happen in S-mode: scause, stvec, sret instead.
```

The PLIC claim/complete protocol is critical: the claim register read acknowledges the interrupt (deasserts the hart's IRQ line), preventing re-entry. The completion write enables the source to interrupt again after it reasserts.

---

## Tier 3 — Advanced

### Question A1
**A system has both M-mode firmware and an S-mode OS. A store instruction in a U-mode process causes a store page fault. Trace the complete event sequence if (a) the page fault is delegated and (b) it is NOT delegated.**

**Answer:**

**(a) Delegated (medeleg[15] = 1):**

```
1. U-mode store executes, MMU page table walk fails (PTE not valid).

2. Hardware (trap to S-mode):
   scause  = 15 (store/AMO page fault)
   sepc    = address of the store instruction
   stval   = faulting virtual address
   sstatus.SPIE = sstatus.SIE; SIE = 0; SPP = 0 (U-mode)
   PC = stvec (kernel page fault handler)

3. S-mode kernel handles the fault:
   - Reads scause (15), stval (faulting address)
   - Determines if the address is a valid mapping for the process
   - If valid (e.g., demand paging): allocates a physical page,
     inserts a PTE, executes SFENCE.VMA to flush TLB
   - Sets sepc to remain at the faulting instruction (not incremented)
   - sret -> U-mode re-executes the store, which now succeeds

4. Total transitions: U -> S -> U (2 privilege changes)
```

**(b) NOT delegated (medeleg[15] = 0):**

```
1. U-mode store executes, MMU page table walk fails.

2. Hardware (trap to M-mode):
   mcause  = 15
   mepc    = address of the store instruction
   mtval   = faulting virtual address
   mstatus.MPIE = MIE; MIE = 0; MPP = 0 (U-mode)
   PC = mtvec (M-mode firmware trap handler)

3. M-mode firmware trap handler:
   - Reads mcause (15) - it is a page fault.
   - Firmware cannot handle this (it has no knowledge of the OS's
     page table management).
   - Firmware must re-inject the trap to S-mode:
     * Write scause = 15
     * Write sepc = mepc (faulting instruction address)
     * Write stval = mtval
     * Set sstatus.SPP = mstatus.MPP (preserve original mode)
     * Set sstatus.SPIE = 0
     * Set sstatus.SIE = 0
     * Set mepc = stvec (kernel's trap entry)
     * Set mstatus.MPP = S-mode
     * mret (this enters S-mode at stvec)

4. S-mode kernel handles as in case (a).

5. After kernel fixes the fault:
   sepc still = faulting store instruction
   sret -> U-mode (SPP=0)

6. Total transitions: U -> M -> S -> U (4 privilege changes)
```

The delegation case requires **half** the privilege transitions — this is why Linux-capable RISC-V systems always configure `medeleg` for page faults. The undelegated path requires the firmware to manually re-inject the exception, which is complex, error-prone, and adds measurable latency to every page fault.

---

### Question A2
**Explain the RISC-V interrupt priority model for a system with both M-mode and S-mode interrupts pending simultaneously. What executes first if the hart is currently in S-mode with `sstatus.SIE = 1`?**

**Answer:**

RISC-V interrupt priority has two components: privilege level and cause code priority within a level.

**Priority rules (highest to lowest):**

```
Within M-mode interrupts: MEI > MSI > MTI > SEI > SSI > STI
(Platform may reorder, but this is the recommended default)

Cross-privilege: M-mode interrupts always preempt S-mode execution,
regardless of sstatus.SIE. M-mode interrupts are enabled by mstatus.MIE,
not sstatus.SIE.
```

**Scenario: S-mode running, `sstatus.SIE = 1`, pending interrupts:**

Assume both `mip.MTIP = 1` (machine timer, not delegated) and `mip.STIP = 1` (supervisor timer, delegated to S-mode) are pending.

```
Check 1: Are M-mode interrupts enabled?
  mstatus.MIE is irrelevant while in S-mode — M-mode interrupts
  preempt lower privilege unconditionally if mie.MTIE = 1.
  Result: machine timer interrupt takes priority.

  Hardware:
    mcause = 0x80000007 (machine timer)
    mepc   = address of interrupted S-mode instruction
    mstatus.MPIE = mstatus.MIE; MIE = 0; MPP = 01 (S-mode)
    PC = mtvec (M-mode firmware)
```

After the M-mode timer handler runs and executes `MRET`:
```
  PC = mepc (back to interrupted S-mode instruction)
  mstatus.MIE = MPIE; MPP = U (reset)
  privilege = S-mode (MPP was S)
```

Now the S-mode timer interrupt (`STIP`) can fire:
```
  sstatus.SIE = 1 and mie.STIE = 1 and mip.STIP = 1
  scause = 0x80000005 (S-mode timer)
  PC = stvec (kernel timer handler)
```

**Key insight:** M-mode interrupts are "above" the privilege hierarchy in interrupt delivery — they can interrupt S-mode and U-mode execution unconditionally (controlled only by `mie` bits). S-mode interrupts only fire in S-mode or U-mode, and require `sstatus.SIE = 1` when in S-mode.

This two-level model lets firmware maintain firm real-time guarantees (via M-mode timer) independently of OS interrupt management.
