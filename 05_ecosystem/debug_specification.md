# Debug Specification

## Prerequisites
- RISC-V privilege architecture: M-mode, S-mode, U-mode
- CSR register file and the `mstatus`, `dcsr`, `dpc` registers
- Basic understanding of JTAG as a physical transport

---

## Concept Reference

### Overview: The RISC-V Debug Specification

The RISC-V External Debug Support specification (commonly called "debug spec", version 1.0 is the current stable release) defines a standard hardware debug interface for RISC-V processors. It is implemented as a separate document from the main ISA specification and is maintained by the RISC-V International Debug Task Group.

The specification defines three major components:

```
1. Debug Module (DM):
   - A hardware block that connects to the processor(s) and the debug transport.
   - Implements halt/resume control, register/memory access, and program buffer execution.

2. Debug Transport Module (DTM):
   - Bridges the DM to a physical transport layer (e.g., JTAG).
   - Exposes the DM's register space via the transport protocol.

3. Debug Mode:
   - A special processor state (more privileged than M-mode for most purposes).
   - The processor enters Debug Mode when halted; executes abstract commands or the program buffer.
```

### Debug Module (DM) Register Space

The DM exposes a set of registers accessible via the DTM. Key DM registers:

```
Address  Register Name          Purpose
-------  --------------------   ------------------------------------------------
0x04     data0 - data11         Data registers: argument passing for abstract cmds
0x10     dmcontrol              Halt/resume, reset, hasel (hart select)
0x11     dmstatus               Status: allhalted, anyhalted, allrunning, version
0x12     hartinfo               Hart capabilities (data registers, ...nscratch)
0x16     abstracts              Abstract command status: busy, cmderr
0x17     command                Execute an abstract command
0x18     abstractauto           Auto-execute on data register access
0x1A     progbuf0 - progbuf15   Program buffer: 16-instruction scratch space
0x40     sbcs                   System bus access control and status
0x41     sbaddress0 - sbaddress3 System bus address
0x48     sbdata0 - sbdata3      System bus data
```

### Halt and Resume

The fundamental debug operation is halting a hart (hardware thread):

```
Halt sequence (debugger to DM):
  1. Write dmcontrol: set haltreq=1 for the target hart (identified by hartsel).
  2. Poll dmstatus until allhalted=1.
  3. Hart is now in Debug Mode: PC is saved to dpc CSR, mstatus saved.

Resume sequence:
  1. Write dmcontrol: set resumereq=1.
  2. Poll dmstatus until allresumed=1 (or allrunning=1).
  3. Hart resumes execution from dpc.
```

When a hart enters Debug Mode:
- The current PC is saved in the `dpc` CSR.
- `dcsr.cause` records why Debug Mode was entered: halt request (3), single-step (4), breakpoint trigger (2), reset-halt (5).
- The hart begins executing at a well-known debug ROM address (`0x800` in the default address map).
- The debug ROM runs a "park loop" waiting for the DM to issue abstract commands.

### Abstract Commands

Abstract commands allow the debugger to read/write registers and memory without the host having direct memory access to the target. The command is written to the `command` DM register:

```
Abstract command register encoding (32-bit):
  [31:24] cmdtype:  0=Access Register, 1=Quick Access, 2=Access Memory
  [23:0]  control:  command-specific fields

Access Register command (cmdtype=0):
  [22:20] aarsize:  2=32-bit, 3=64-bit, 4=128-bit
  [19]    aarpostincrement: auto-increment for sequential access
  [17]    transfer: 1=perform the register transfer
  [16]    write:    0=read register->data0, 1=write data0->register
  [15:0]  regno:    0x0000-0x0FFF = CSRs, 0x1000-0x101F = GPRs x0-x31, 0x1020+ = FP/etc.

Example: Read GPR x3 (regno = 0x1003):
  Write command = 0x00231003
    cmdtype=0, aarsize=3 (64-bit), transfer=1, write=0, regno=0x1003
  After command completes: GPR x3 value is in data0 (and data1 for 64-bit)
```

**Abstract command error handling:**
If an abstract command fails, the `abstractcs.cmderr` field is set:
- 0: no error
- 1: busy (previous command still running)
- 2: not supported
- 3: exception (command caused a RISC-V exception)
- 4: halt/resume error
- 7: other error

### Program Buffer

The program buffer is a small writable instruction memory inside the DM (typically 2-16 instructions, 16 in the full implementation). The debugger writes RISC-V instructions to `progbuf0`-`progbuf15`, then issues a `command` with `postexec=1` to execute them on the halted hart.

```
Program buffer use case: read an arbitrary CSR (e.g., mcycle)

1. Write to progbuf0: CSRR x8, mcycle   (0xB0041473)
   Write to progbuf1: EBREAK            (0x00100073) -- return to debug mode
2. Issue abstract command: Access Register, transfer=1, write=0, regno=0x1008 (x8)
   But first, execute the progbuf:
   Set command = {cmdtype=0, postexec=1, transfer=0, ...}
3. After execution: x8 = mcycle value
4. Issue another abstract command to read x8 -> data0

Alternatively, use the Abstract Access Memory command or combine steps.
```

The program buffer enables access to any operation the RISC-V ISA can express — floating-point registers, vector registers, custom CSRs — without needing specific DM support for each.

### Trigger Module: Hardware Breakpoints and Watchpoints

The RISC-V debug spec defines a trigger module with configurable hardware triggers. Each trigger can be configured as:

```
tselect CSR: select which trigger to configure (0 to NTRIGGERS-1)
tdata1 CSR:  trigger type and control
tdata2 CSR:  trigger address/data match value
tdata3 CSR:  additional matching (context, chain)

Trigger types (tdata1[63:60] = type):
  2 = Address/Data Match (mcontrol):
       - Match on instruction fetch (breakpoint), load address, store address, or data value
       - Match modes: equal, NAPOT (naturally aligned power of 2), range

  6 = Interrupt Trigger (icount):
       - Trigger after N instructions have executed (single-step variant)

  4 = Interrupt on specific interrupt:
       - Trigger on specific interrupt sources

mcontrol (tdata1, type=2) key fields:
  [63:60] type   = 2
  [59]    dmode  = 1 (trigger only accessible in Debug Mode -- prevents software tampering)
  [21]    m      = 1 enable in M-mode
  [19]    s      = 1 enable in S-mode
  [18]    u      = 1 enable in U-mode
  [17]    execute= 1 match on instruction fetch (hardware breakpoint)
  [16]    store  = 1 match on store address/data (watchpoint)
  [15]    load   = 1 match on load address/data (watchpoint)
  [10:7]  match  = 0 (exact address match), 1 (NAPOT)
  [6]     chain  = 1 (chain this trigger with the next for range matching)
```

### OpenOCD and the RISC-V Debug Interface

OpenOCD (Open On-Chip Debugger) is the standard open-source tool that implements the host side of the RISC-V debug protocol. It connects to the target via a hardware JTAG adapter and presents a GDB Remote Serial Protocol (RSP) interface to GDB.

```
Debugging stack:

  Developer machine:
    GDB (riscv64-unknown-elf-gdb)
      |  GDB Remote Serial Protocol (RSP) over TCP port 3333
      v
    OpenOCD
      |  JTAG commands over USB
      v
    JTAG adapter (J-Link, FTDI, CMSIS-DAP, etc.)
      |  TCK, TMS, TDI, TDO pins
      v
    DTM in the SoC (translates JTAG to DM register accesses)
      |  internal debug bus
      v
    Debug Module (DM)
      |  debug request signals
      v
    RISC-V Hart(s)
```

**OpenOCD configuration file example (`riscv_debug.cfg`):**
```tcl
# OpenOCD configuration for a RISC-V SoC with FTDI JTAG adapter

interface ftdi
ftdi_vid_pid 0x0403 0x6010
ftdi_layout_init 0x0008 0x001b
ftdi_layout_signal nSRST -oe 0x0020 -data 0x0020

transport select jtag

set _CHIPNAME riscv
jtag newtap $_CHIPNAME cpu -irlen 5

set _TARGETNAME $_CHIPNAME.cpu
target create $_TARGETNAME riscv -chain-position $_TARGETNAME

# RISC-V-specific OpenOCD settings:
riscv set_prefer_sba on    # use system bus access for memory (faster than progbuf)
riscv set_command_timeout_sec 2
```

**GDB session example:**
```bash
# Terminal 1: start OpenOCD
openocd -f riscv_debug.cfg

# Terminal 2: GDB
riscv64-unknown-elf-gdb ./firmware.elf
(gdb) target extended-remote :3333
(gdb) load                          # flash the binary
(gdb) monitor reset halt            # reset and halt
(gdb) break main
(gdb) continue
(gdb) print/x $a0                   # print register a0 in hex
(gdb) x/10i $pc                     # disassemble 10 instructions at PC
(gdb) watch *0x20000000             # set a watchpoint
```

---

## Tier 1 — Fundamentals

### Question F1
**What are the three main components of the RISC-V external debug specification, and what does each do?**

**Answer:**

**1. Debug Module (DM):**
The DM is a hardware block inside the SoC that provides control and access to the RISC-V harts. It implements:
- Halt and resume control: the debugger sends halt/resume requests; the DM signals these to each hart.
- Register and memory access via abstract commands: the debugger reads/writes GPRs, CSRs, and memory without needing a running program.
- Program buffer: a small scratchpad of instruction memory that the halted hart can execute, enabling access to resources (like FP registers) that abstract commands do not directly support.
- System bus access: direct bus access to SoC memory without using the processor at all.

**2. Debug Transport Module (DTM):**
The DTM is the bridge between the physical transport layer and the DM's register space. Most RISC-V implementations use JTAG as the transport, with a DTM that maps the JTAG TAP data register to DM register read/write operations. The DTM is typically a small logic block that:
- Implements the JTAG TAP (Test Access Port) state machine.
- Provides a DMI (Debug Module Interface) — a simple address/data/op bus.
- Translates JTAG shift operations into DMI transactions to the DM.

**3. Debug Mode:**
Debug Mode is a special processor state, distinct from M/S/U privilege modes, that the hart enters when halted by the debugger. It is "more privileged" than M-mode in the sense that:
- Debug Mode can access any CSR regardless of the hart's normal privilege state.
- `dcsr.stopcount` can prevent cycle/instruction counters from incrementing.
- Debug Mode can bypass PMP and PMA checks (configurable).
- The hart executes at a special debug ROM address, not at the normal PC.

Debug Mode allows the debugger to inspect and modify the full architectural state without requiring the normal trap-handling infrastructure to be functional (useful for debugging early boot or bare-metal code).

---

### Question F2
**What is a hardware breakpoint in the context of the RISC-V debug specification, and how does it differ from a software breakpoint (`EBREAK` instruction)?**

**Answer:**

**Software breakpoint:**
A software breakpoint is implemented by replacing the instruction at the target address with an `EBREAK` instruction (encoding `0x00100073`). When execution reaches that address, `EBREAK` causes a breakpoint exception, which is routed to the debugger (via Debug Mode trigger, or via the trap handler if debugging via semihosting).

Limitations:
- Requires write access to the instruction memory (cannot be used in ROM or flash that cannot be re-programmed at runtime).
- Uses instruction memory bandwidth.
- Cannot break on data access (watchpoints) — only on instruction execution.
- The original instruction must be saved and restored when the breakpoint is removed.

**Hardware breakpoint (trigger):**
A hardware breakpoint uses the RISC-V trigger module. The `tdata2` CSR is loaded with the target address; `tdata1.execute` is set to 1; `tdata1.m/s/u` enable the trigger in the desired privilege modes. When the hart fetches an instruction at `tdata2`, the trigger fires before execution and the hart enters Debug Mode.

Advantages over software breakpoints:
- Works in ROM, flash, and any read-only memory.
- The instruction at the breakpoint address is not modified.
- Can be set on data addresses (store/load watchpoints) by setting `tdata1.store` or `tdata1.load` instead of `execute`.
- More suitable for debugging time-sensitive or interrupt-driven code where modifying instructions could change timing.

Limitations:
- The number of hardware triggers is fixed by the implementation (typically 2-8). A processor with 4 triggers supports at most 4 hardware breakpoints simultaneously.
- Software breakpoints are unlimited (subject only to instruction memory size).

In practice, debuggers like GDB use hardware breakpoints preferentially when available, falling back to software breakpoints when all hardware triggers are in use.

---

### Question F3
**What is the `dcsr` (Debug Control and Status Register) and what information does it contain when a hart enters Debug Mode?**

**Answer:**

`dcsr` is a Debug Mode CSR (accessible only in Debug Mode) that records why the hart entered Debug Mode and controls Debug Mode behaviour.

**Key fields:**

```
dcsr fields [RISC-V Debug Spec]:

  [31:28] xdebugver: debug spec version (4 = 1.0)
  [15]    ebreakm:   1 = EBREAK in M-mode enters Debug Mode (not M-mode exception)
  [14]    ebreaks:   1 = EBREAK in S-mode enters Debug Mode
  [13]    ebreaku:   1 = EBREAK in U-mode enters Debug Mode
  [11]    stepie:    1 = interrupts enabled during single-step
  [10]    stopcount: 1 = minstret/mcycle do not increment in Debug Mode
  [9]     stoptime:  1 = time-related CSRs do not advance in Debug Mode
  [8:6]  cause:     reason for entering Debug Mode:
                      1 = ebreak (software breakpoint or EBREAK instruction)
                      2 = trigger (hardware breakpoint/watchpoint)
                      3 = haltreq (debugger requested halt)
                      4 = step (single-step completed)
                      5 = resethaltreq (halt on reset was requested)
  [2]    mprven:    1 = mstatus.MPRV effective in Debug Mode
  [1]    step:      1 = single-step mode enabled (enter Debug Mode after each instruction)
  [1:0]  prv:       privilege mode the hart was in when it entered Debug Mode
                      0=U, 1=S, 3=M
```

**On entering Debug Mode:**

`dcsr.cause` is updated to indicate why Debug Mode was entered. The debugger reads this to distinguish:
- A halt request (cause=3): the user clicked "pause" in the IDE.
- A breakpoint hit (cause=2): a hardware trigger fired.
- A single-step completion (cause=4): one instruction executed after `dcsr.step=1`.

`dcsr.prv` records the privilege level so that, on resume, the hart returns to the correct privilege mode.

`dpc` (a separate register) holds the PC at which Debug Mode was entered. The debugger reads `dpc` to determine where execution was when halted, and writes `dpc` to redirect execution when resuming (e.g., to skip over a breakpoint instruction).

---

## Tier 2 — Intermediate

### Question I1
**Walk through the exact sequence of DM register reads and writes that a debugger (OpenOCD) performs to read the value of register `x10` (a0) from a halted RV64 hart using an abstract command.**

**Answer:**

**Preconditions:** The hart is already halted (dmstatus.allhalted = 1). Hart 0 is selected (dmcontrol.hartsel = 0).

**Step 1: Verify the hart is halted and idle**
```
Read  dmstatus (addr 0x11)
  Check: allhalted = 1 (the hart is halted)

Read  abstractcs (addr 0x16)
  Check: busy = 0 (no abstract command in progress)
  Check: cmderr = 0 (no previous error)
```

**Step 2: Issue the abstract command to read GPR x10**
```
GPR x10 has regno = 0x1000 + 10 = 0x100A

Abstract command register value:
  cmdtype  [31:24] = 0      (Access Register command)
  aarsize  [22:20] = 3      (64-bit access for RV64)
  postexec [18]    = 0      (no program buffer execution)
  transfer [17]    = 1      (perform the register transfer)
  write    [16]    = 0      (read: register -> data0/data1)
  regno    [15:0]  = 0x100A (x10)

command value = 0x00230_100A = 0x0023100A

Write command (addr 0x17) = 0x0023100A
```

**Step 3: Wait for completion**
```
Poll abstractcs (addr 0x16):
  Read abstractcs; spin while busy = 1.
  Timeout after implementation-defined period (e.g., 100 ms).

After busy = 0:
  Check cmderr != 0 -> if error, the command failed:
    cmderr = 2: command not supported (unlikely for GPR access)
    cmderr = 3: exception during execution (rare for halted hart)
  Clear cmderr by writing abstractcs with cmderr field set (W1C).
```

**Step 4: Read the result from data0 and data1**
```
Read  data0 (addr 0x04) -> lower 32 bits of x10
Read  data1 (addr 0x05) -> upper 32 bits of x10

64-bit value of x10 = (data1 << 32) | data0
```

**Total DM transactions:** 5-7 (2 status reads, 1 command write, 1-2 polls, 2 data reads). Each DMI transaction involves a JTAG shift, so at typical JTAG speeds (10 MHz), each transaction takes ~5 µs. Total register read time: ~35 µs. This is why JTAG debugging feels slow compared to native debugging — every register read requires multiple bus transactions.

---

### Question I2
**Explain the program buffer and give a concrete example of how it is used to access floating-point register f5 on a RISC-V hart that supports the F extension.**

**Answer:**

The program buffer is a block of writable instruction memory inside the Debug Module, typically 16 32-bit words. The DM exposes these as registers `progbuf0`-`progbuf15`. The halted hart can be made to execute these instructions by issuing an abstract command with `postexec=1`, or by issuing the abstract command with `cmdtype=0` and using the program buffer to move values to/from GPRs.

**Why the program buffer is needed for FP registers:**

Abstract commands with `cmdtype=0` (Access Register) directly support GPRs (regno 0x1000-0x101F) and CSRs (regno 0x0000-0x0FFF). FP registers are at regno 0x1020-0x103F. Some DM implementations support these directly; others require the program buffer to move the FP value to a GPR first.

**Example: Read f5 using the program buffer**

```
Step 1: Write program buffer
  progbuf0 = FMV.X.W x8, f5     -- move f5 (float) to x8 (integer), bit-for-bit
                                    encoding: 0xE0240453
                                    (funct7=0x70, rs1=f5=5, funct3=0, rd=x8=8, opcode=0x53)
  progbuf1 = EBREAK              -- return to debug mode
                                    encoding: 0x00100073

  Write progbuf0 (addr 0x20) = 0xE0240453
  Write progbuf1 (addr 0x21) = 0x00100073

Step 2: Execute the program buffer AND transfer x8
  Abstract command:
    cmdtype  = 0   (Access Register)
    aarsize  = 2   (32-bit, single-precision float)
    postexec = 0   (execute first, then transfer)
    transfer = 1   (read: register -> data0)
    write    = 0
    regno    = 0x1008  (x8)

  BUT we need to execute the progbuf FIRST, then read x8.
  Method: issue command with postexec=1 and transfer=0 to execute the progbuf only:

  First command: execute progbuf (no register transfer)
    command = {cmdtype=0, aarsize=2, postexec=1, transfer=0, write=0, regno=0}
    Write command (addr 0x17) = 0x00240000 (with postexec=1 bit set)
    Poll abstractcs until busy=0

  Second command: read x8 (no progbuf execution)
    command = {cmdtype=0, aarsize=2, postexec=0, transfer=1, write=0, regno=0x1008}
    Write command = 0x00230008 (transfer=1, x8)
    Poll abstractcs until busy=0
    Read data0 (addr 0x04) -> 32-bit float value of f5
```

**For RV64 with double-precision (D extension), FMV.X.D is used instead:**
```
  progbuf0 = FMV.X.D x8, f5   -- 64-bit float to integer, encoding: 0xE2240453
  aarsize  = 3                 -- 64-bit transfer
  Result: data0 (lower 32 bits) and data1 (upper 32 bits) = 64-bit double value
```

**Efficiency note:** Each access to an FP register via the program buffer requires 5-8 DM transactions. Accessing a single GPR directly requires 3-4 transactions. For bulk register dumps, the DM's `abstractauto` register can be set to automatically issue repeat commands on each data register read, reducing the per-register overhead to 2-3 transactions.

---

## Tier 3 — Advanced

### Question A1
**A hardware engineer is designing a RISC-V SoC with a custom debug module. The DM must support: (a) halting all 4 harts simultaneously, (b) accessing each hart's CSRs independently, and (c) a 4-instruction program buffer. Describe the key design decisions for the DM hardware, focusing on the hart selection mechanism, the abstract command state machine, and the program buffer arbitration.**

**Answer:**

**Part (a): Halting all 4 harts simultaneously**

The `dmcontrol` register has a `hasel` (hart array select) bit that switches between single-hart selection (hasel=0, using `hartsel`) and multi-hart selection (hasel=1, using the Hart Array Window). The Hart Array Window (`hawindowsel` and `hawindow` DM registers) allows the debugger to write a bitmask of hart indices to halt.

Hardware design:
```
dmcontrol.haltreq must fan out to all selected harts.
Each hart has a dedicated "halt request" input signal from the DM.

Multi-halt logic:
  if hasel == 0:
      assert halt_req[hartsel]     -- single hart
  if hasel == 1:
      for each hart i:
          if hawindow[i] == 1:
              assert halt_req[i]   -- all selected harts

Simultaneous halt:
  All harts receive halt_req in the same clock cycle.
  Each hart independently checks halt_req and enters Debug Mode at the next instruction boundary.
  Harts may not halt in the exact same cycle (pipeline depth variation), but the
  spec allows "any halt" and "all halt" status bits in dmstatus to distinguish.
```

**Part (b): Independent CSR access per hart**

Each hart has its own CSR file. The program buffer approach (writing `CSRR`/`CSRW` instructions and executing them on the target hart) is the most portable. The key hardware consideration:

- The DM must know which hart will execute the program buffer.
- `dmcontrol.hartsel` selects the hart; the DM must route the "execute program buffer" signal only to that hart's debug ROM.
- Abstract command status (`abstractcs.cmderr`) must be per-hart or have a mechanism to clear per-hart errors.

For designs where direct CSR access (without program buffer) is desired, the DM can expose a dedicated CSR bus:
```
DM -> CSRR bus -> arbiter -> per-hart CSR file
                  (one hart at a time, selected by hartsel)
```

**Part (c): 4-instruction program buffer arbitration**

The program buffer is a shared SRAM accessible by both the DTM (for writes from the debugger) and the halted hart's instruction fetch unit (for execution).

Hardware design:
```
Program buffer SRAM:
  Size: 4 * 32 bits = 128 bits
  Read port: connected to the halted hart's instruction fetch path
             (at the special debug ROM address range, e.g., 0x80000-0x8000F)
  Write port: connected to the DM register write path (from DTM)

Arbitration:
  The DM state machine controls access:
  - In IDLE state: DTM write port is active; hart cannot execute.
  - When abstract command with postexec is issued:
      DM transitions to EXECUTE state.
      DTM write port is disabled (or writes are ignored).
      Hart's instruction fetch is directed to the program buffer address.
  - EBREAK at end of program buffer: hart re-enters Debug Mode;
      DM transitions back to IDLE.
      Hart instruction fetch returns to normal debug ROM (park loop).

Abstract command state machine:
  IDLE -> (command write) -> CHECKING (verify command valid)
  CHECKING -> (valid) -> TRANSFER (for Access Register commands)
  CHECKING -> (postexec) -> EXECUTING (run program buffer)
  TRANSFER -> (done) -> IDLE
  EXECUTING -> (EBREAK trap) -> IDLE
  Any state -> (exception) -> ERROR (set cmderr, transition to IDLE)
```

**Critical timing consideration:** The program buffer execution must complete before the debugger can issue another abstract command. The `abstractcs.busy` bit is the synchronisation mechanism. The DM asserts `busy=1` from the moment the command is written until execution completes and results are stable in data0. Polling busy at the JTAG speed (one poll per ~5-10 µs) creates a minimum command latency; for 4-instruction program buffers at 100 MHz, execution takes ~40 ns — far less than the JTAG poll interval. The busy logic is therefore straightforward: deassert after the EBREAK exception is taken.

---

### Question A2
**OpenOCD reports "Error: abstract command execution failed, cmderr=3 (exception)" when attempting to read a CSR via the program buffer. Explain the possible causes, how to diagnose the root cause using the RISC-V debug spec, and how to recover.**

**Answer:**

`cmderr=3` (exception) means that while the DM was executing an abstract command (using the program buffer), the RISC-V hart raised an exception. The abstract command halted abnormally.

**Possible causes:**

1. **Illegal instruction exception (accessing a CSR that does not exist or is not accessible in Debug Mode):**
   ```
   Program buffer: CSRR x8, <csr_addr>
                   EBREAK
   If <csr_addr> does not exist on this implementation, the CSRR raises an
   illegal instruction exception (mcause=2).
   ```

2. **CSR access privilege violation:**
   ```
   Some CSRs are read-only or require a specific privilege level.
   Example: Writing mcycle (0xB00) via CSRW in the program buffer may raise
   an illegal instruction if mcycle is hardwired (read-only) on this implementation.
   ```

3. **Access to a disabled extension's CSR:**
   ```
   If mstatus.FS = Off (FP disabled), accessing any FP CSR (fflags, frm, fcsr)
   raises an illegal instruction exception.
   ```

4. **Memory access fault in the program buffer:**
   ```
   If the program buffer contains a load/store, and the target address is invalid
   (unmapped, PMP-protected), a load/store access fault is raised.
   ```

**Diagnosis procedure:**

```
Step 1: Read abstractcs to get cmderr (already known: 3 = exception).
        Write abstractcs to clear cmderr (write 1s to cmderr field, W1C).

Step 2: Read DPC (using abstract command to read dpc CSR, regno 0x07B1).
        DPC tells you which instruction in the program buffer caused the exception.
        If DPC = progbuf_base + N, instruction N caused the exception.

Step 3: Read mcause (regno 0x0342) to identify the exception type.
        2 = illegal instruction (wrong CSR address, disabled extension)
        5 = load access fault
        7 = store access fault

Step 4: Read mtval (regno 0x0343) for additional context.
        For illegal instructions: mtval = the faulting instruction encoding.
        For memory faults: mtval = the faulting address.

Step 5: Inspect mstatus (regno 0x0300) to check if FP is enabled (mstatus.FS != 0)
        if the failure involves FP CSR access.
```

**Recovery procedure:**

```
1. Clear cmderr by writing abstractcs with cmderr=7 (write 1s, W1C).
   Write abstractcs (addr 0x16) = 0x00000700

2. If the cause was a disabled extension:
   a. Use a program buffer sequence to enable the extension first:
      progbuf0: CSRS mstatus, (MSTATUS_FS_INITIAL << FS_SHIFT)  -- enable FP
      progbuf1: EBREAK
   b. Issue execute-only command to run this program buffer.
   c. Then retry the original CSR access.

3. If the cause was a non-existent CSR:
   a. Consult the implementation's CSR list (hardware documentation or
      RISC-V misa register to determine which extensions are implemented).
   b. Do not attempt to access unimplemented CSRs.
   c. Use the trigger module's dmode bit to verify which triggers are available
      before attempting tselect/tdata operations.

4. If the program buffer caused a memory fault:
   a. Verify the target address is valid using DM system bus access (sbcs/sbaddress/sbdata)
      to probe the address independently before executing program buffer loads/stores.
   b. Check PMP configuration (read pmpcfg0 and pmpaddr0-N CSRs).
```

The general recovery rule: after any `cmderr != 0`, clear the error before issuing further abstract commands, or the DM will reject new commands with `cmderr=1` (busy/error state).
