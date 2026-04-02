# Privilege Modes: M, S, U

## Overview

RISC-V defines up to three privilege levels — Machine (M), Supervisor (S), and User (U) — that form a concentric ring model. Each level controls which instructions are legal, which CSRs are accessible, and which memory regions are reachable. Understanding how privilege levels transition and interact is fundamental to writing firmware, operating system kernels, and hypervisors on RISC-V.

---

## Key Concepts

### The Three Privilege Levels

RISC-V encodes the current privilege level in a two-bit field that is not directly accessible to software but is reflected in several CSR fields (notably `mstatus.MPP` and `mstatus.SPP`).

| Level | Encoding | Typical Use |
|---|---|---|
| User (U) | `00` | Application code |
| Supervisor (S) | `01` | OS kernel, hypervisor guest |
| Machine (M) | `11` | Firmware, bootloader, SEE |

Encoding `10` is reserved (used for Hypervisor H-mode in extensions).

Not every implementation provides all three levels. A minimal embedded system may implement M-mode only. An application processor running Linux requires at least M and S; U is needed to isolate user-space applications.

### Machine Mode (M-mode)

M-mode is the highest privilege level and is mandatory on every RISC-V implementation. It is the first mode entered after reset.

Responsibilities:
- Platform initialisation (clock setup, DRAM training, device tree creation)
- Trap handling for exceptions and interrupts that are not delegated to S-mode
- Configuring physical memory protection (PMP) regions
- Providing the Supervisor Execution Environment (SEE) interface — the M-mode firmware that S-mode calls via `ecall`

M-mode has unrestricted access to all CSRs and all physical memory (subject only to PMP checks configured in M-mode, and even those can be bypassed for M-mode itself unless the lock bit is set).

### Supervisor Mode (S-mode)

S-mode is used by operating system kernels. It sits below M-mode but above U-mode.

Responsibilities:
- Managing virtual memory (`satp` CSR, page tables)
- Handling traps delegated from M-mode (via `medeleg`/`mideleg`)
- Scheduling and isolating U-mode processes
- Responding to U-mode `ecall` system call requests

S-mode cannot access M-mode CSRs (attempting to do so raises an illegal instruction exception). S-mode sees a subset of the full CSR space: `sstatus`, `stvec`, `sepc`, `scause`, `stval`, `satp`, `sip`, `sie`, and a few others.

### User Mode (U-mode)

U-mode is the least privileged level, used for application code.

U-mode cannot:
- Access any S- or M-mode CSRs
- Execute privileged instructions (`mret`, `sret`, `wfi`, `sfence.vma`, etc.)
- Access physical memory directly (virtual memory is enforced by the MMU controlled from S-mode)

U-mode requests OS services via the `ecall` instruction, which triggers an environment call exception. The trap is taken either to S-mode (if delegated) or M-mode.

### Privilege Level Transitions

Mode transitions occur via controlled mechanisms — arbitrary jumps between privilege levels are not possible.

**Entering higher privilege (trap entry):**
Any exception or interrupt causes the processor to switch to either M-mode or S-mode depending on the delegation configuration. The processor:
1. Saves the current PC in `mepc` or `sepc`
2. Saves the cause in `mcause` or `scause`
3. Saves the current privilege level in `mstatus.MPP` or `mstatus.SPP`
4. Disables interrupts by clearing `mstatus.MIE` or `mstatus.SIE`
5. Jumps to the address in `mtvec` or `stvec`

**Returning to lower privilege (trap return):**
`MRET` (M-mode) and `SRET` (S-mode) are the trap-return instructions. They:
1. Restore the PC from `mepc` or `sepc`
2. Restore the privilege level from `mstatus.MPP` or `mstatus.SPP`
3. Re-enable interrupts by restoring `mstatus.MPIE` or `mstatus.SPIE` into `MIE`/`SIE`
4. Set `MPP` to U (or least-privileged mode) to prevent privilege escalation on the next trap

**Voluntary privilege elevation:**
U-mode calls S-mode via `ecall` (environment call). S-mode calls M-mode via `ecall`. These generate an environment call exception, not a direct branch.

```
         ┌────────────┐
 Reset ──►   M-mode   │  mret ──► S-mode or U-mode
         │  (firmware)│◄──────── ecall from S-mode
         └────────────┘
               │ mret (sets MPP=S)
               ▼
         ┌────────────┐
         │   S-mode   │  sret ──► U-mode
         │  (kernel)  │◄──────── ecall/exception from U-mode (if delegated)
         └────────────┘
               │ sret (sets SPP=U)
               ▼
         ┌────────────┐
         │   U-mode   │
         │   (app)    │
         └────────────┘
```

### Privilege Level in mstatus

The `mstatus` CSR tracks privilege information across trap boundaries:

- `MPP` (bits 12:11): Previous privilege mode before the last M-mode trap
- `SPP` (bit 8): Previous privilege mode before the last S-mode trap (1 = S, 0 = U)
- `MIE` (bit 3): M-mode interrupt enable
- `SIE` (bit 1): S-mode interrupt enable
- `MPIE` (bit 7): Saved MIE before the last M-mode trap
- `SPIE` (bit 5): Saved SIE before the last S-mode trap

When a trap is taken to M-mode: MIE is saved to MPIE, MIE is cleared, current privilege is saved to MPP, and mode switches to M.

When `MRET` executes: MIE is restored from MPIE, MPIE is set to 1, mode switches to the value in MPP, and MPP is set to U.

### WFI and Privilege Restrictions

`WFI` (Wait For Interrupt) is a privileged hint instruction. It is legal in M-mode and S-mode. In U-mode it raises an illegal instruction exception unless `mstatus.TW` (Timeout Wait) is cleared, in which case it is treated as a NOP.

This allows M-mode firmware to decide whether U-mode code is permitted to issue WFI (useful for RTOS tasks that manage their own sleep).

---

## Interview Questions

### Fundamentals Tier

---

**Q1. What are the three RISC-V privilege modes, and what is each used for?**

**Answer:**

- **Machine mode (M-mode)**: The highest privilege level, mandatory on all RISC-V implementations. Used for firmware, bootloaders, and the Supervisor Execution Environment. The processor starts in M-mode after reset. M-mode has unrestricted access to all hardware resources.

- **Supervisor mode (S-mode)**: An intermediate level used by operating system kernels. S-mode manages virtual memory and handles traps delegated from M-mode. It cannot access M-mode CSRs.

- **User mode (U-mode)**: The least privileged level for application code. U-mode cannot access privileged CSRs or execute privileged instructions. It interacts with the kernel via the `ecall` instruction.

---

**Q2. How does a RISC-V processor transition from U-mode to S-mode?**

**Answer:**

The processor cannot jump directly from U-mode to S-mode. The only path is through a trap (exception or interrupt):

1. A U-mode program executes `ecall` (environment call exception), generates a page fault, or an interrupt fires.
2. The hardware checks whether the trap is delegated to S-mode via `medeleg` (exceptions) or `mideleg` (interrupts). If delegated, the trap is taken to S-mode.
3. The processor saves the current PC in `sepc`, the cause in `scause`, the faulting address (if any) in `stval`, and the current privilege (U=0) in `mstatus.SPP`.
4. Interrupts are disabled by clearing `mstatus.SIE`.
5. The PC jumps to the address in `stvec`.

Return to U-mode happens via `SRET`, which restores the PC from `sepc` and the privilege from `mstatus.SPP`.

---

**Q3. What happens on reset in RISC-V? Which mode does the processor start in?**

**Answer:**

On reset, the RISC-V processor starts in **M-mode**. This is guaranteed by the specification regardless of which privilege levels the implementation supports.

After reset:
- The PC is set to an implementation-defined reset vector (typically in ROM or flash)
- M-mode CSRs have their reset values (implementation-defined, but `mstatus.MIE` = 0, interrupts disabled)
- No virtual memory translation is active
- The firmware (M-mode code) initialises hardware, configures PMP, sets up `mtvec`, optionally sets up `medeleg`/`mideleg`, and eventually jumps to S-mode or U-mode by setting `mstatus.MPP` and executing `MRET`

---

**Q4. What is the `ecall` instruction and what does it do?**

**Answer:**

`ecall` (Environment Call) is an I-type instruction (opcode `SYSTEM`, funct3=0, funct12=0) that generates a synchronous exception, transferring control to the trap handler at a higher privilege level.

- From U-mode: generates exception cause `Environment call from U-mode` (cause 8). Typically delegated to S-mode for POSIX system calls.
- From S-mode: generates exception cause `Environment call from S-mode` (cause 9). Used to request services from M-mode firmware (via the SBI — Supervisor Binary Interface).
- From M-mode: generates exception cause `Environment call from M-mode` (cause 11). Taken by the M-mode trap handler itself.

The calling convention uses `a0`–`a7` for arguments and return values. The specific ABI is defined by the SBI specification (S-to-M calls) or the OS ABI (U-to-S calls).

---

**Q5. Can U-mode code read or write M-mode CSRs? What happens if it tries?**

**Answer:**

No. Any attempt by U-mode code to access an M-mode or S-mode CSR raises an **illegal instruction exception** (`mcause` = 2). The access is blocked before the CSR is read or written.

CSR access permission is encoded in the top two bits of the 12-bit CSR address:
- Bits 11:10 encode the minimum privilege required to access the CSR:
  - `00` or `01` = U-mode accessible
  - `10` = S-mode or above
  - `11` = M-mode only
- Bit 10 and 11 also encode whether the CSR is read-only (if both are 1).

The hardware compares the current privilege level against the CSR address encoding on every CSR instruction.

---

**Q6. What is `MRET` and when is it used?**

**Answer:**

`MRET` (Machine Return) is a privileged instruction used by M-mode trap handlers to return from a trap. It is the only architectural way to exit an M-mode trap.

What `MRET` does atomically:
1. Sets the PC to the value in `mepc`
2. Sets the current privilege level to the value in `mstatus.MPP`
3. Sets `mstatus.MIE` to `mstatus.MPIE` (restores interrupt enable state)
4. Sets `mstatus.MPIE` to 1
5. Sets `mstatus.MPP` to U (or the least-privileged mode supported)

M-mode firmware uses `MRET` at boot time to transition to S-mode (kernel) by:
- Placing the kernel entry point in `mepc`
- Setting `mstatus.MPP` = 1 (S-mode)
- Executing `MRET`

Without `MRET`, there is no way for M-mode code to voluntarily drop to a lower privilege level.

---

### Intermediate Tier

---

**Q7. Explain the role of `mstatus.MPP` and `mstatus.SPP`. What values do they hold and when?**

**Answer:**

`MPP` (Machine Previous Privilege, bits 12:11 of `mstatus`) records the privilege level that was active immediately before the most recent trap taken to M-mode. `SPP` (Supervisor Previous Privilege, bit 8) records the privilege level before the most recent trap to S-mode.

| Field | Width | Encoding | Capture time | Restore by |
|---|---|---|---|---|
| MPP | 2 bits | 00=U, 01=S, 11=M | On any M-mode trap | `MRET` |
| SPP | 1 bit  | 0=U, 1=S | On any S-mode trap | `SRET` |

SPP is only 1 bit because only U or S can trap into S-mode (M-mode traps go to M-mode, not S-mode).

The trap-return instructions use these fields to restore the previous privilege, then reset the field to U to enforce the principle of least privilege: if another trap occurs before the field is written by the trap handler, the return from that trap defaults to U-mode rather than an elevated mode.

A critical security implication: `SRET` executed in S-mode with `SPP`=1 returns to S-mode (a horizontal transition). This is used when S-mode handles a nested trap and then returns to the same privilege level.

---

**Q8. What is the Supervisor Binary Interface (SBI), and how does it use RISC-V privilege modes?**

**Answer:**

The SBI is a standardised interface between S-mode software (OS kernels) and M-mode firmware (OpenSBI, BIOS). It is analogous to BIOS/UEFI calls on x86 or ARM's PSCI.

The SBI uses `ecall` from S-mode to M-mode. By convention:
- `a7` holds the SBI extension ID
- `a6` holds the function ID
- `a0`–`a5` hold arguments
- `a0` returns an error code, `a1` returns a value

Example SBI calls:
- `sbi_console_putchar` (legacy extension 0x01) — print a character to debug console
- `sbi_hart_start` / `sbi_hart_stop` — start/stop a hardware thread (HSM extension 0x48534D)
- `sbi_send_ipi` — send inter-processor interrupt
- `sbi_set_timer` — program the `mtimecmp` register (which S-mode cannot access directly)

The SBI decouples the OS from specific M-mode hardware details. A Linux kernel compiled for RISC-V issues `sbi_set_timer` without knowing whether the underlying machine is a HiFive Unmatched, a QEMU virtual machine, or a custom SoC — OpenSBI handles the hardware-specific register write.

---

**Q9. How does a RISC-V processor handle a trap that occurs while already in M-mode?**

**Answer:**

By default, if a trap occurs while the processor is already in M-mode (a **re-entrant trap**), the trap is still taken to M-mode. This overwrites `mepc`, `mcause`, `mstatus.MPIE`, and `mstatus.MPP`.

The consequences are significant: the previous trap's return information is lost unless the trap handler explicitly saves `mepc` and `mstatus` to the stack before re-enabling interrupts.

The standard pattern in M-mode trap handlers:
```asm
# M-mode trap entry
csrrw  t0, mscratch, t0    # swap t0 with pointer to trap frame
sw     ra,  0(t0)          # save all registers
sw     a0,  4(t0)
# ... save all caller/callee registers ...
csrr   a0, mepc            # SAVE mepc before re-enabling interrupts
sw     a0, FRAME_EPC(t0)
csrsi  mstatus, MSTATUS_MIE  # re-enable M-mode interrupts (allows nesting)
# ... handle trap ...
lw     a0, FRAME_EPC(t0)
csrw   mepc, a0            # RESTORE mepc before mret
# ... restore registers ...
mret
```

If `mstatus.MIE` is left cleared throughout the handler (the simpler approach), re-entrant traps cannot occur and `mepc` is safe. Most simple firmware takes this approach.

---

**Q10. What is the `TVM`, `TW`, and `TSR` bits in `mstatus`, and how do they let M-mode restrict S-mode behaviour?**

**Answer:**

These three bits in `mstatus` are "trap" control bits that allow M-mode to intercept specific S-mode operations, enabling M-mode to emulate or override functionality:

- **TVM (Trap Virtual Memory, bit 20)**: When set, any S-mode attempt to read or write `satp` or execute `SFENCE.VMA` generates an illegal instruction exception taken to M-mode. Use case: M-mode implements a Type-1 hypervisor that manages a second level of page tables and needs to intercept all VM context switches.

- **TW (Timeout Wait, bit 21)**: When set, `WFI` executed in S-mode (or U-mode if TW was set before delegation) will raise an illegal instruction exception after an implementation-defined timeout. Use case: Hypervisor environments where the M-mode hypervisor needs to prevent a guest OS from putting the hart to sleep without its permission.

- **TSR (Trap SRET, bit 22)**: When set, executing `SRET` in S-mode raises an illegal instruction exception to M-mode. Use case: Hypervisor intercepting supervisor-level trap returns to manage a guest's mode transitions.

Together these bits allow M-mode firmware to act as a lightweight type-1 hypervisor without full H-extension hardware support, at the cost of emulation overhead for every intercepted operation.

---

**Q11. Describe the boot sequence on a typical RISC-V Linux platform (e.g., HiFive Unmatched).**

**Answer:**

1. **Reset**: All harts start in M-mode at the platform's reset vector (often a ZSBL — Zero Stage Boot Loader in ROM).

2. **ZSBL / First Stage Boot Loader**: Minimal M-mode code that copies the second-stage bootloader from flash to DRAM and jumps to it.

3. **OpenSBI (M-mode firmware)**: Takes over in M-mode. Initialises M-mode CSRs, configures PMP to protect its own memory, sets `mtvec` to its own trap handler, programs `medeleg`/`mideleg` to delegate most exceptions and interrupts to S-mode, then uses `mret` to jump to U-Boot or the kernel in S-mode.

4. **U-Boot or kernel (S-mode)**: The OS bootloader or kernel starts in S-mode. It configures `satp` to enable virtual memory, sets `stvec` for its own trap handler, and begins the kernel boot sequence.

5. **User-space (U-mode)**: After the kernel boots, it creates user processes by configuring page tables and using `sret` to drop to U-mode at the entry point of the first user-space program (typically `/sbin/init`).

OpenSBI remains resident in M-mode memory throughout, handling SBI calls from the kernel via `ecall` from S-mode.

---

### Advanced Tier

---

**Q12. What privilege mode security vulnerabilities can arise from incorrect `MPP` handling, and how does RISC-V prevent privilege escalation?**

**Answer:**

Incorrect `MPP` handling can cause **privilege escalation** — a trap handler that returns to a higher privilege than the code that triggered the trap.

**Scenario: forgetting to set MPP before MRET**

If M-mode firmware sets up a new execution context (e.g., loads a kernel) but forgets to set `mstatus.MPP` before executing `MRET`, the processor returns to M-mode (the reset default), and the "kernel" code runs with full machine privileges. Any kernel vulnerability then has complete hardware access.

**RISC-V's architectural defence:**
After every `MRET`, the hardware **sets MPP to U** (or the least-privileged supported mode). This means if a second trap immediately fires after `MRET` before the handler can set MPP, the second trap-return defaults to U-mode — never escalating silently.

**Scenario: malicious S-mode setting SPP=S before SRET**

S-mode code could write `mstatus.SPP = 1` then execute `SRET`, returning to S-mode instead of U-mode. This is not a privilege escalation (S returns to S), but it defeats the isolation of U-mode processes. RISC-V allows S-mode to write `SPP` — it is the kernel's responsibility to set `SPP = 0` before returning to user space. A kernel bug here causes a user process to continue running in S-mode.

**Best practice in firmware:** Always explicitly set `mstatus.MPP` to the target privilege level and zero out register state before every `MRET` that switches privilege.

---

**Q13. Explain how a Type-1 hypervisor can be implemented using only M-mode on a RISC-V without H-extension, and what its limitations are.**

**Answer:**

Without the H-extension, M-mode can act as a type-1 hypervisor by using the TVM, TW, and TSR bits to intercept guest OS operations:

**Architecture:**
- M-mode = hypervisor
- Guest OS runs in S-mode (thinking it is the OS)
- Guest applications run in U-mode

**Intercepting VM operations:**
- Set `mstatus.TVM = 1`: any guest OS access to `satp` traps to M-mode. M-mode emulates the `satp` write by constructing a shadow page table that enforces physical memory partitioning between VMs.
- Set `mstatus.TSR = 1`: any guest `SRET` traps to M-mode, allowing the hypervisor to manage guest mode transitions.
- `medeleg`/`mideleg`: most exceptions are *not* delegated — they go to M-mode for the hypervisor to inspect and forward to the guest S-mode handler.

**Limitations:**
1. **Performance**: Every guest `satp` write, `SFENCE.VMA`, and `SRET` traps to M-mode. Modern OS kernels issue these frequently — context switches, TLB flushes. The trap overhead is ~hundreds of cycles per operation.
2. **Two VMs sharing one hart**: Only one VM's context can be active at a time. Switching VMs requires saving/restoring all S-mode CSRs.
3. **No two-level address translation**: With H-extension, hardware does guest-physical to host-physical translation automatically. Without it, M-mode must maintain and update shadow page tables, which is complex and expensive.
4. **No guest interrupt virtualization**: Direct guest interrupt delivery requires H-extension's HVIP register. Without it, M-mode must emulate all interrupt delivery.

This approach works for simple partitioning (e.g., a real-time partition + Linux partition on an embedded SoC) but does not scale to cloud-style multi-VM workloads.

---

**Q14. What is the RISC-V H-extension, and how does it extend the privilege model beyond M/S/U?**

**Answer:**

The H (Hypervisor) extension adds hardware virtualisation support, introducing two new privilege modes:

- **HS-mode (Hypervisor-extended Supervisor mode)**: A hypervisor running at the same hardware privilege as S-mode, but with access to additional H-mode CSRs. Replaces S-mode in a virtualised system.
- **VS-mode (Virtual Supervisor mode)**: A guest OS runs here, thinking it is in S-mode. It has its own set of VS-level CSRs (`vsstatus`, `vstvec`, `vsepc`, etc.) that shadow the real S-mode CSRs.
- **VU-mode (Virtual User mode)**: Guest user applications run here, thinking they are in U-mode.

The privilege hierarchy becomes: M > HS > (VS, VU), with U still accessible from HS.

**Two-level address translation (G-stage):**
HS-mode configures a second set of page tables (`hgatp`) that translate guest-physical addresses to host-physical addresses. VS-mode manages its own page tables for guest-virtual to guest-physical (VS-stage). The hardware walks both levels transparently.

**VS-mode CSR aliases:**
When running in VS-mode, reads/writes to `sstatus` actually access `vsstatus`; `stvec` accesses `vstvec`, etc. This means a guest OS does not need modification — its CSR accesses work as expected, but they are redirected to per-VM state.

**Key new CSRs for HS-mode:**
- `hgatp`: host-level G-stage page table base
- `hedeleg` / `hideleg`: delegate exceptions/interrupts from HS-mode to VS-mode
- `hvip`: inject virtual interrupts into VS-mode
- `hstatus`: track VS-mode privilege information

The H-extension eliminates the need for shadow page tables and the overhead of TVM-trapping, making it practical to run multiple full OS instances on a single RISC-V chip.
