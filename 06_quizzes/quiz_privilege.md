# Quiz: Privilege Architecture

## Instructions

15 multiple-choice questions covering RISC-V privilege modes, CSR registers, trap
handling, virtual memory (Sv32/Sv39), and physical memory protection (PMP). Each
question has exactly one correct answer. Work through all questions before checking
the answer key at the end.

Difficulty distribution: Questions 1-5 Fundamentals, Questions 6-11 Intermediate,
Questions 12-15 Advanced.

---

## Questions

### Q1 (Fundamentals)

How many privilege levels does the RISC-V specification define (including those added
by the H extension), and which one is mandatory on every implementation?

- A) Two levels (M and U); M-mode is mandatory.
- B) Three levels (M, S, U); all three are mandatory.
- C) Three base levels (M, S, U) plus two virtualisation levels (VS, VU); M-mode is
     mandatory and is the only one required by every implementation.
- D) Four levels (M, S, U, H); S-mode is mandatory for any OS-capable system.

---

### Q2 (Fundamentals)

What privilege mode does a RISC-V processor always start in after reset?

- A) User mode (U-mode)
- B) Supervisor mode (S-mode)
- C) Machine mode (M-mode)
- D) The reset mode is implementation-defined and cannot be assumed.

---

### Q3 (Fundamentals)

Which CSR holds the exception program counter — the address the processor returns to
after completing a machine-mode trap handler?

- A) `mtvec`
- B) `mcause`
- C) `mepc`
- D) `mtval`

---

### Q4 (Fundamentals)

What does the `ecall` instruction do when executed in U-mode?

- A) It directly switches the processor into S-mode and continues execution at `stvec`.
- B) It generates a synchronous environment-call exception, causing a trap to either
     S-mode (if delegated via `medeleg`) or M-mode.
- C) It sends a software interrupt to the next hart in the hart list.
- D) It reads a value from the system control block and places it in `a0`.

---

### Q5 (Fundamentals)

In the RISC-V physical memory protection scheme, what is the maximum number of PMP
regions that the specification supports?

- A) 8
- B) 16
- C) 64
- D) 256

---

### Q6 (Intermediate)

The CSR `mstatus` contains the field `MPP` (bits 12:11). What does `MPP` record, and
when is it written by hardware?

- A) The current privilege level; written on every clock cycle.
- B) The privilege level that was active before the most recent M-mode trap; written
     automatically by hardware when a trap is taken to M-mode.
- C) The target privilege level to switch to on the next `MRET`; written by software.
- D) The privilege level of the last instruction that wrote to a CSR; written by the
     CSR access hardware.

---

### Q7 (Intermediate)

A RISC-V platform has M-mode firmware and an S-mode OS. Which CSR pair does the
firmware write to delegate exceptions and interrupts from M-mode down to S-mode?

- A) `mtvec` and `stvec`
- B) `medeleg` and `mideleg`
- C) `mcause` and `scause`
- D) `mie` and `sie`

---

### Q8 (Intermediate)

Sv39 is the 39-bit virtual memory scheme for RV64. How many levels does its page
table have, and what is the size of each page at the leaf level?

- A) Two levels; 4 KB pages.
- B) Three levels; 4 KB pages at the leaf (with optional 2 MB and 1 GB huge pages at
     higher levels).
- C) Four levels; 4 KB pages.
- D) Three levels; 8 KB pages to match the 64-bit PTE width.

---

### Q9 (Intermediate)

A PMP entry is configured with address matching mode `NAPOT` and `pmpaddr0 = 0x0200_001F`.
What physical address range does this protect?

- A) A single 4-byte word at address `0x0800_007C`.
- B) A 256-byte region starting at `0x0800_0000`.
- C) A 64-byte region starting at `0x0800_0000`.
- D) An 8-byte naturally aligned region starting at address derived from `pmpaddr0`.

---

### Q10 (Intermediate)

The `mtvec` CSR encodes the trap vector base address and a mode field in bits [1:0].
What is the difference between mode `00` (Direct) and mode `01` (Vectored)?

- A) Direct mode: all traps jump to the base address. Vectored mode: interrupts jump to
     `BASE + 4 * cause`, allowing each interrupt cause to have its own handler entry
     point without a software dispatch table.
- B) Direct mode: only synchronous exceptions use `mtvec`; interrupts use a separate
     CSR. Vectored mode: both exceptions and interrupts use `mtvec + 4 * cause`.
- C) Direct mode: the full 32-bit vector address is in `mtvec`. Vectored mode: only the
     upper 20 bits are stored; the lower 12 bits are zero.
- D) Direct and Vectored modes behave identically for exceptions; they differ only in
     how the processor handles nested traps.

---

### Q11 (Intermediate)

When an S-mode OS writes to `satp` to install a new page table, what additional
operation must it perform to ensure the hardware TLB does not serve stale translations?

- A) Write to the `mstatus.SUM` bit to flush cached permission checks.
- B) Execute `SFENCE.VMA` (with appropriate `rs1` and `rs2` arguments) to invalidate
     TLB entries for the affected address space.
- C) Execute `FENCE.I` to flush the instruction cache, which also clears TLB state.
- D) Write the new `satp` value twice; the second write signals the hardware to flush.

---

### Q12 (Advanced)

A RISC-V Sv39 page table walk for virtual address `0xFFFF_FFFF_8000_0000` on an RV64
system. The `satp` register holds `MODE=8` (Sv39) and `PPN=0x80400`. Describe the
first step of the hardware page table walk: which physical address is the root page
table located at, and which bits of the virtual address select the VPN[2] index?

- A) Root page table at `0x8040_0000`; VPN[2] from VA bits [38:30] = `0x1FF` (511).
- B) Root page table at `0x8040_0000`; VPN[2] from VA bits [47:39] = `0x1FF` (511).
- C) Root page table at `0x8040_0_000`; VPN[2] from VA bits [38:30] = `0x100` (256).
- D) Root page table at `0x4020_0000`; VPN[2] from VA bits [38:30] = `0x1FF` (511).

---

### Q13 (Advanced)

Explain the effect of the `mstatus.MPRV` bit. Under what microarchitectural scenario
is it useful, and what happens to memory accesses when it is set?

- A) MPRV enables M-mode to bypass PMP checks entirely, allowing firmware to access any
     physical address without raising access faults.
- B) When MPRV=1, load and store instructions use the privilege mode in `mstatus.MPP`
     for memory access permission checks (virtual memory and PMP), rather than the
     current M-mode privilege. This allows M-mode firmware to emulate memory accesses
     on behalf of a lower-privilege mode.
- C) MPRV makes virtual memory translation optional for S-mode; the OS can disable
     address translation for specific stores by setting MPRV.
- D) MPRV redirects all M-mode CSR accesses to S-mode shadow CSRs, enabling a
     hypervisor to intercept firmware CSR operations.

---

### Q14 (Advanced)

A PMP region is configured with the `L` (lock) bit set. What are the two effects of
the lock bit, and why is this important for security?

- A) The lock bit makes the PMP entry read-only from S-mode; M-mode can still modify it
     via a special unlock sequence.
- B) Setting the lock bit does two things: (1) it prevents further modification of that
     PMP entry (including from M-mode) until the next reset, and (2) it enforces the
     PMP permission check on M-mode accesses as well as lower-privilege accesses.
     Without the lock bit, M-mode is always exempt from PMP checks.
- C) The lock bit enables the PMP entry to apply across privilege level boundaries,
     including to hypervisor-extended S-mode (HS-mode) memory accesses.
- D) Setting the lock bit forces the entry into TOR (Top Of Range) address matching
     mode, overriding any previously configured matching mode.

---

### Q15 (Advanced)

A trap is taken to M-mode while the processor is already executing in M-mode (a
re-entrant trap). What critical state is at risk, and what must an M-mode trap handler
do to support safe nested traps?

- A) The `mscratch` register is overwritten by hardware on every trap entry; the
     handler must save and restore it from a fixed memory address before re-enabling
     interrupts.
- B) The values in `mepc`, `mcause`, and `mstatus.MPIE`/`mstatus.MPP` are overwritten
     by the new trap, destroying the previous trap's return context. The handler must
     save `mepc` and the relevant `mstatus` fields to a stack frame before re-enabling
     interrupts (by setting `mstatus.MIE`), otherwise a nested trap will corrupt the
     outer handler's ability to execute `MRET` correctly.
- C) Re-entrant traps in M-mode are architecturally prevented; the processor queues the
     second trap and delivers it only after the first `MRET` is executed.
- D) Only `mepc` is overwritten; `mcause` and `mstatus` are banked per trap level and
     are restored automatically when the inner trap handler executes `MRET`.

---

## Answer Key

| Q  | Answer | Difficulty    |
|----|--------|---------------|
| 1  | C      | Fundamentals  |
| 2  | C      | Fundamentals  |
| 3  | C      | Fundamentals  |
| 4  | B      | Fundamentals  |
| 5  | C      | Fundamentals  |
| 6  | B      | Intermediate  |
| 7  | B      | Intermediate  |
| 8  | B      | Intermediate  |
| 9  | B      | Intermediate  |
| 10 | A      | Intermediate  |
| 11 | B      | Intermediate  |
| 12 | A      | Advanced      |
| 13 | B      | Advanced      |
| 14 | B      | Advanced      |
| 15 | B      | Advanced      |

---

## Detailed Explanations

### Q1 - Answer: C

The base spec defines three privilege levels: M (Machine), S (Supervisor), and U (User).
The H (Hypervisor) extension adds VS-mode (Virtual Supervisor) and VU-mode (Virtual User),
bringing the total to five. M-mode is the only level that is architecturally mandatory;
a minimal embedded core may implement M-mode only, omitting S and U entirely.

- **A** is wrong: two levels exist in minimal embedded implementations, but the spec
  formally defines three base levels plus two H-extension levels.
- **B** is wrong: all three base levels are not mandatory — S and U are optional.
- **D** is wrong: H is not a privilege level itself; it is an extension that adds VS
  and VU. S-mode is not mandatory.

---

### Q2 - Answer: C (Machine mode)

The RISC-V privileged spec mandates that every hart begins execution in M-mode after
reset. The reset vector address is implementation-defined (often pointing to ROM or
flash), but the privilege level is always M. This allows M-mode firmware to perform
trusted hardware initialisation before any lower-privilege code runs.

- **A and B** are wrong: starting in a lower privilege mode would mean that untrusted
  code runs before the hardware is configured — a security and correctness problem.
- **D** is wrong: the specification explicitly requires M-mode at reset; it is not
  implementation-defined.

---

### Q3 - Answer: C (`mepc`)

`mepc` (Machine Exception Program Counter) saves the PC value that was current when
the trap was taken. For synchronous exceptions, `mepc` holds the address of the
faulting instruction. For interrupts, it holds the address of the next instruction
to be executed after the interrupt is serviced. `MRET` restores the PC from `mepc`.

- **A (`mtvec`)** holds the trap handler base address — the destination of the trap,
  not the return address.
- **B (`mcause`)** encodes the reason for the trap (exception code or interrupt bit).
- **D (`mtval`)** holds additional trap information such as the faulting virtual address
  for a page fault, or the instruction encoding for an illegal instruction exception.

---

### Q4 - Answer: B

`ecall` generates a synchronous environment-call exception. The hardware consults
the delegation registers: if the corresponding bit in `medeleg` is set, the trap is
taken to S-mode (via `stvec`); otherwise it is taken to M-mode (via `mtvec`). The
processor never jumps directly to a lower or equal privilege level without going
through the full trap mechanism.

- **A** is wrong: `ecall` does not directly switch modes; the mode switch is a
  consequence of the trap machinery.
- **C** is wrong: `ecall` is a local synchronous exception, not an inter-hart message.
- **D** is wrong: `ecall` does not read from a system control block; it generates a
  trap and transfers control to the registered handler.

---

### Q5 - Answer: C (64)

The RISC-V privileged spec allows up to 64 PMP entries, numbered `pmp0cfg` through
`pmp63cfg`, with corresponding `pmpaddr0` through `pmpaddr63`. Each entry covers one
physical region. Implementations may provide any number from 0 to 64 in multiples of
4 (since four entries are packed into each `pmpcfg` CSR).

- **A (8)** is a common minimum implementation but not the specification maximum.
- **B (16)** is another common implementation size but still not the maximum.
- **D (256)** exceeds what the spec defines.

---

### Q6 - Answer: B

`MPP` (Machine Previous Privilege) is written automatically by hardware when any trap
is taken to M-mode. It captures the two-bit privilege encoding of the mode that was
active at the point of the trap (`00`=U, `01`=S, `11`=M). `MRET` reads `MPP` to
determine which privilege mode to restore, then resets `MPP` to U.

- **A** is wrong: `MPP` is not the current privilege level; that is an internal
  hardware state not directly readable as a single field.
- **C** is wrong: while software can write `MPP` to control where `MRET` will return,
  the hardware writes `MPP` automatically on trap entry, which is the primary mechanism.
- **D** is wrong: CSR access privilege is encoded in the CSR address itself, not in
  `mstatus.MPP`.

---

### Q7 - Answer: B (`medeleg` and `mideleg`)

`medeleg` (Machine Exception Delegation) and `mideleg` (Machine Interrupt Delegation)
are the CSRs that allow M-mode to delegate specific exception causes and interrupt
causes to S-mode. When bit N is set in `medeleg`, exception cause N is handled by
S-mode's trap handler (via `stvec`, `sepc`, `scause`, `stval`). This is how a Linux
kernel receives page faults, system calls from U-mode, etc., without involving M-mode
firmware for every trap.

- **A** is wrong: `mtvec` and `stvec` are the handler address registers, not the
  delegation registers.
- **C** is wrong: `mcause` and `scause` record the cause after the trap has been taken;
  they do not control delegation.
- **D** is wrong: `mie` and `sie` are interrupt-enable masks, not delegation controls.

---

### Q8 - Answer: B (3 levels, 4 KB leaf pages)

Sv39 uses a 3-level radix page table. Each virtual address is split into three 9-bit
VPN fields (VPN[2], VPN[1], VPN[0]) and a 12-bit page offset. Leaf PTEs at level 0
map 4 KB pages. Non-leaf PTEs at level 1 can be leaf PTEs mapping 2 MB huge pages,
and non-leaf PTEs at level 2 can map 1 GB gigapages, giving three page size options.

- **A** is wrong: that describes Sv32 (the RV32 scheme), which has two levels.
- **C** is wrong: Sv48 uses four levels; Sv39 uses three.
- **D** is wrong: the page offset in Sv39 is 12 bits (4096 bytes = 4 KB), not 13 bits.

---

### Q9 - Answer: B (256-byte region starting at `0x0800_0000`)

NAPOT (Naturally Aligned Power Of Two) encoding: the address in `pmpaddr` encodes
both the base address and the size. The number of trailing ones in the address
determines the size: `0x0200_001F` in binary ends with `11111` (5 trailing ones),
so the region size is `2^(5+3) = 256` bytes. The base address is obtained by clearing
the trailing ones and their surrounding bit: `0x0200_001F & ~0x1F = 0x0200_0000`,
then shifting left by 2 (since `pmpaddr` stores PA[55:2]): `0x0200_0000 << 2 =
0x0800_0000`.

- **A** is wrong: NAPOT address encoding is not a byte pointer to a 4-byte word.
- **C** is wrong: 4 trailing ones give 64 bytes; 5 trailing ones give 256 bytes.
- **D** is wrong: the 8-byte option would require 0 trailing ones (`0x...00`), but
  that is actually the NA4 (Naturally Aligned 4-byte) mode, not NAPOT.

---

### Q10 - Answer: A

In Direct mode (`mtvec[1:0] = 00`), all traps (both exceptions and interrupts) jump
to the address `mtvec[MXLEN-1:2] << 2` — a single entry point. The handler must read
`mcause` to dispatch to the right sub-handler in software.

In Vectored mode (`mtvec[1:0] = 01`), synchronous exceptions still jump to the base
address, but asynchronous interrupts jump to `BASE + 4 * cause_code`. This allows
the interrupt controller to route each interrupt directly to its handler without a
software dispatch table, reducing interrupt latency.

- **B** is wrong: in both modes, exceptions go to the base address; the difference is
  only in interrupt routing.
- **C** is wrong: both modes store the full base address in the upper bits of `mtvec`;
  the lower two bits are the mode field, not missing address bits.
- **D** is wrong: the modes have a clear and documented difference for interrupts; they
  do not behave identically.

---

### Q11 - Answer: B (`SFENCE.VMA`)

The TLB caches virtual-to-physical address translations. Writing `satp` changes the
active page table root, but hardware TLBs do not automatically flush on `satp` writes
(doing so on every context switch would be extremely expensive). The software must
explicitly execute `SFENCE.VMA` to invalidate TLB entries. The `rs1` argument
specifies the virtual address to invalidate (or all addresses if `rs1=x0`), and `rs2`
specifies the ASID (or all ASIDs if `rs2=x0`).

- **A** is wrong: `mstatus.SUM` (Supervisor User Memory) controls whether S-mode can
  access U-mode memory pages; it has nothing to do with TLB flushing.
- **C** is wrong: `FENCE.I` flushes the instruction cache / instruction fetch pipeline;
  it does not affect the data TLB.
- **D** is wrong: there is no architectural mechanism where writing `satp` twice
  triggers a flush; that is not in the specification.

---

### Q12 - Answer: A

For Sv39, the physical address of the root page table is `PPN << 12`. With `PPN =
0x80400`, the root is at `0x80400 << 12 = 0x8040_0000`.

The virtual address `0xFFFF_FFFF_8000_0000` in Sv39 is a 39-bit sign-extended address.
VPN[2] is extracted from VA bits [38:30]. For this address:
- bits [38:30] of `0xFFFF_FFFF_8000_0000` = `1_1111_1111` = `0x1FF` = 511.

The root page table entry at index 511 is read from physical address
`0x8040_0000 + 511 * 8 = 0x8040_0FF8`.

- **B** is wrong: bits [47:39] are above the 39-bit virtual address width in Sv39;
  they must be sign extensions of bit 38 and are not part of the VPN.
- **C** is wrong: `0x100` (256) would come from a different virtual address; the
  calculation of VPN[2] from this specific address yields 511.
- **D** is wrong: the root page table physical address calculation `PPN << 12` with
  `PPN = 0x80400` gives `0x8040_0000`, not `0x4020_0000`.

---

### Q13 - Answer: B

`mstatus.MPRV` (Modify PRiVilege) alters how load and store instructions check memory
permissions when the processor is in M-mode. Normally, M-mode accesses are checked
against M-mode privilege (bypassing PMP unless the L bit is set). With MPRV=1, loads
and stores use the privilege level in `mstatus.MPP` for their permission checks instead.

A primary use case: M-mode firmware needs to copy data to or from a U-mode buffer on
behalf of a system call. With MPRV=1 and MPP set to U, the firmware's store will fail
with an access fault if the U-mode pointer is invalid — exactly the security check that
should be applied. Without MPRV, M-mode would silently succeed even on bad pointers.

- **A** is wrong: MPRV does not bypass PMP; it changes which privilege level is used
  for the permission check, potentially making M-mode more restricted, not less.
- **C** is wrong: MPRV applies to M-mode loads/stores, not to S-mode translation.
- **D** is wrong: MPRV has no interaction with CSR access or hypervisor shadow CSRs.

---

### Q14 - Answer: B

The PMP lock bit (`pmpcfg.L = 1`) has two distinct and important effects:

1. **Immutability**: The `pmpcfg` and `pmpaddr` entries for that region become
   read-only even to M-mode. Writes are silently ignored. The lock is cleared only by
   a system reset. This prevents M-mode firmware from accidentally (or maliciously)
   disabling a security boundary at runtime.

2. **M-mode enforcement**: Normally, M-mode accesses are exempt from PMP checks.
   With the lock bit set, the PMP entry applies to M-mode accesses as well. This is
   essential for a TEE (Trusted Execution Environment): the secure firmware uses a
   locked PMP entry to protect its own memory from being read or written even by other
   M-mode code that may be compromised.

- **A** is wrong: the lock bit applies to M-mode too and there is no unlock sequence
  other than reset.
- **C** is wrong: the lock bit has no interaction with H-extension HS-mode semantics.
- **D** is wrong: the lock bit does not change the address matching mode.

---

### Q15 - Answer: B

When a trap is taken to M-mode, hardware automatically overwrites `mepc` with the
new faulting PC, `mcause` with the new cause, and updates `mstatus.MPIE` and
`mstatus.MPP` with the current interrupt-enable and privilege state. If the processor
is already in M-mode handling a trap and a second trap fires (for example, because the
handler re-enabled interrupts with `csrsi mstatus, MSTATUS_MIE`), all of these CSRs
are overwritten and the original return context is lost.

The solution is to save `mepc`, `mcause`, and the relevant `mstatus` fields to a
stack frame (in the trap frame pointed to by `mscratch`) before setting `mstatus.MIE`
to allow nesting. The corresponding restore before `MRET` then works correctly.

- **A** is wrong: `mscratch` is a general-purpose scratch register for software use;
  hardware does not write to it on trap entry.
- **C** is wrong: RISC-V does not queue or defer re-entrant M-mode traps; a second
  trap taken to M-mode immediately overwrites the M-mode trap CSRs.
- **D** is wrong: RISC-V does not bank `mcause` or `mstatus` per trap nesting level;
  there is only one copy of each M-mode CSR.
