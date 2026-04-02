# Simulation: Spike and QEMU

## Prerequisites
- RISC-V ISA fundamentals (RV32I/RV64I, extensions, privilege modes)
- Basic understanding of ELF binaries and the RISC-V toolchain
- Familiarity with the RISC-V privilege architecture (M, S, U modes; trap handling)

---

## Concept Reference

### Functional vs. Performance Simulation

Before covering specific tools, it is important to distinguish the two major simulation categories:

```
Functional simulation:
  - Models WHAT instructions do (register updates, memory changes, exceptions)
  - Does not model WHEN (no clock cycle accuracy)
  - Fast: can simulate billions of instructions per second
  - Used for: software development, ISA compliance testing, co-simulation reference

Performance simulation:
  - Models timing: pipeline stages, cache behaviour, branch prediction
  - Cycle-accurate or event-driven
  - Slow: typically 1-100 million simulated cycles per second on the host
  - Used for: architecture exploration, microarchitecture design, power estimation
```

Spike is a functional simulator. QEMU is a dynamic binary translation (DBT) functional emulator. Neither is a cycle-accurate performance simulator. For RISC-V cycle-accurate simulation, tools like Verilator (RTL simulation), gem5, or FireSim (FPGA-based) are used.

### Spike: The RISC-V ISA Simulator

Spike (riscv-isa-sim) is the official RISC-V ISA golden reference simulator, maintained at `github.com/riscv-software-src/riscv-isa-sim`. It is written in C++ and implements the RISC-V ISA specification exactly as written.

**Key characteristics:**
- Interprets RISC-V instructions one by one (not compiled/translated).
- Implements all standard extensions: RV32/64 I, M, A, F, D, C, V, plus privilege architecture.
- Can run bare-metal programs (no OS), programs under a proxy kernel (pk), or full Linux via BBL.
- Provides an interactive debugger mode.
- Used as the reference for RISC-V compliance tests.

**Running a bare-metal binary:**
```bash
# Compile a RISC-V bare-metal program
riscv64-unknown-elf-gcc -march=rv64imac -mabi=lp64 -o hello hello.c

# Run under Spike with the proxy kernel (pk handles syscalls: printf, exit, etc.)
spike pk hello

# Run with a specific ISA configuration
spike --isa=rv32imac pk hello_rv32

# Spike with log output (trace every instruction)
spike -l pk hello 2>spike_trace.log
```

**Proxy kernel (pk):**
The proxy kernel is a minimal supervisor-mode OS that runs under Spike. It forwards RISC-V system calls to the host OS via a host call (HTIF — Host Target Interface). This means programs that use `printf`, `malloc`, `exit`, and standard file I/O work without a full OS.

```
Host machine (x86 Linux)
  |
  +-- Spike (RISC-V ISA simulator)
        |
        +-- pk (RISC-V proxy kernel, runs in S-mode)
              |
              +-- hello (RISC-V application, runs in U-mode)
                    | printf() -> RISC-V libc -> ecall -> pk -> HTIF -> host printf
```

**Spike interactive debug mode:**
```bash
spike -d pk hello   # -d enters interactive debug mode

# Spike debugger commands:
(spike) reg 0 a0       # show register a0 of hart 0
(spike) mem 0 0x80001000  # show memory at address
(spike) until pc 0 0x80001234  # run until PC = address
(spike) run 100        # execute 100 instructions
(spike) step           # single-step one instruction
```

**Custom extension simulation in Spike:**
Spike's instruction decoding is table-driven. Adding a custom instruction requires:
1. Writing a C++ class derived from `insn_t` that implements the instruction semantics.
2. Registering the instruction in the decode table via a Spike plugin or fork.

```cpp
// Spike custom instruction example (simplified):
// Custom instruction: SADD rd, rs1, rs2  -- saturating add
// Encoding: custom-0 opcode space (bits [6:0] = 0x0B)

DEFINE_INSN(sadd) {
    sreg_t a = RS1;         // read rs1
    sreg_t b = RS2;         // read rs2
    sreg_t sum = a + b;
    // Saturate on overflow
    if (a > 0 && b > 0 && sum < 0) sum = INT64_MAX;
    if (a < 0 && b < 0 && sum > 0) sum = INT64_MIN;
    WRITE_RD(sum);
}
```

### HTIF: Host Target Interface

HTIF is the communication channel between Spike (the target) and the host machine. It allows the simulated RISC-V code to perform I/O and to signal termination:

```
HTIF mechanism:
  - Two magic memory-mapped locations: tohost and fromhost
  - Writing to tohost signals a host call request
  - The host reads tohost, performs the requested action, and writes to fromhost

  tohost = 0x1 -> signal exit with code 0 (used by tests to indicate pass)
  tohost = (code << 1) | 1 -> exit with non-zero code (test failure)
  tohost = syscall_packet -> proxy syscall (printf, file I/O, etc.)

In ELF binaries: tohost and fromhost are global symbols in the .tohost section.
Spike detects these symbols and monitors their values.
```

### QEMU: Quick Emulator

QEMU (`qemu-system-riscv64` and `qemu-riscv64`) provides RISC-V emulation using dynamic binary translation (DBT). Unlike Spike's interpretation, QEMU compiles guest RISC-V instructions into host machine code (x86_64) at runtime, executing the compiled host code directly. This gives significantly higher speed than Spike.

**Two QEMU modes:**

```
1. System mode (qemu-system-riscv64):
   - Emulates a complete hardware platform: CPU, RAM, peripherals, UART, interrupt controller.
   - Can boot a full Linux kernel + rootfs.
   - Models machine-level (M-mode) privilege; the guest kernel runs in S-mode.
   - Used for: full OS testing, kernel development, SoC bringup simulation.

2. User mode (qemu-riscv64):
   - Emulates only the CPU (U-mode instructions).
   - Intercepts system calls and translates them to host syscalls.
   - Faster than system mode (no peripheral emulation).
   - Used for: running Linux ELF binaries cross-compiled for RISC-V on an x86 host.
```

**QEMU system mode example:**
```bash
# Boot a minimal RISC-V Linux image in QEMU (virt machine)
qemu-system-riscv64 \
  -machine virt \
  -nographic \
  -kernel fw_jump.elf \
  -device loader,file=u-boot.bin,addr=0x80200000 \
  -drive file=rootfs.ext2,format=raw,id=hd0 \
  -device virtio-blk-device,drive=hd0 \
  -append "root=/dev/vda rw console=ttyS0"

# Machine: virt -- a generic RISC-V virtual platform
# fw_jump.elf: OpenSBI firmware (M-mode, starts S-mode at 0x80200000)
# u-boot.bin: U-Boot bootloader (S-mode)
```

**QEMU user mode example:**
```bash
# Run a RISC-V Linux binary on an x86 host
qemu-riscv64 ./my_riscv_program

# With sysroot (find shared libraries)
qemu-riscv64 -L /opt/riscv/sysroot ./my_riscv_program

# With GDB server for debugging
qemu-riscv64 -g 1234 ./my_riscv_program &
riscv64-unknown-linux-gnu-gdb -ex "target remote :1234" my_riscv_program
```

**QEMU virt machine memory map (relevant for bare-metal development):**
```
0x0000_0000 - 0x0000_0FFF:  Debug ROM
0x0000_2000 - 0x0000_3FFF:  Test/control registers (HTIF-like for testing)
0x0002_0000 - 0x0002_FFFF:  Boot ROM (contains device tree blob)
0x0C00_0000 - 0x0FFF_FFFF:  PLIC (Platform-Level Interrupt Controller)
0x1000_0000 - 0x1000_00FF:  UART0 (16550-compatible)
0x8000_0000 - 0xFFFF_FFFF:  DRAM (up to 2 GB by default)
```

### Co-Simulation: Spike + RTL

A common verification methodology uses Spike as a "golden reference" alongside an RTL simulation (Verilator or VCS):

```
Co-simulation flow:

  1. Run the same instruction stream on both Spike and the RTL design.
  2. After each instruction, compare architectural state:
     - PC, all registers, CSRs.
  3. If state diverges: the RTL has a bug (or Spike has a bug, but that is rare).

Implementation:
  - Spike provides a C API (libriscv.a) for step-by-step simulation.
  - The RTL simulation wrapper calls Spike's step() function and compares outputs.
  - On mismatch: dump the divergence point, the instruction, and the register file.

Tools that implement this:
  - riscv-dv (Google): generates random instruction streams and co-simulates
  - Chipyard: RISC-V SoC framework with Spike co-simulation built in
  - CVA6 (Ariane) verification infrastructure
```

### Comparison: Spike vs. QEMU

| Feature                    | Spike                        | QEMU (system)              |
|----------------------------|------------------------------|----------------------------|
| Primary use                | ISA reference, compliance    | OS booting, software dev   |
| Execution model            | Interpretation               | Dynamic binary translation |
| Speed (simulated MIPS)     | ~10-50 MIPS                  | ~500-2000 MIPS             |
| ISA accuracy               | Exact (golden reference)     | Very high (minor exceptions)|
| Privilege mode support     | Full M/S/U                   | Full M/S/U                 |
| Extension support          | All standard extensions      | Most standard extensions   |
| Custom instruction support | Plugin/fork                  | Plugin/TCG                 |
| Interactive debugger       | Built-in (-d flag)           | GDB stub (-gdb flag)       |
| Full OS boot               | Via BBL/OpenSBI (slow)       | Yes, primary use case      |
| Peripheral models          | Minimal (CLINT, PLIC, UART)  | Rich (virtio, PCI, network)|

---

## Tier 1 — Fundamentals

### Question F1
**What is the difference between a functional simulator and a cycle-accurate simulator? When would you use each for RISC-V development?**

**Answer:**

A **functional simulator** models the architectural effect of each instruction — what happens to registers, memory, and CSRs — without tracking the timing (how many cycles each operation takes). Spike is a functional simulator: it executes RISC-V instructions and produces exactly the architectural state changes defined by the ISA specification.

A **cycle-accurate simulator** additionally models the microarchitecture — pipeline stages, cache behaviour, memory latency, branch prediction — and reports exactly how many cycles each instruction takes. An RTL simulation of a RISC-V design in Verilator or a SystemC performance model is cycle-accurate.

**When to use functional simulation:**
- Software development and debugging: compiling applications, testing algorithm correctness.
- ISA compliance testing: verifying that an instruction sequence produces the correct architectural output.
- Firmware and OS development: functional correctness of device drivers, trap handlers, boot sequences.
- Early-stage architecture exploration: quickly running workloads to check that the ISA is sufficient.
- Speed-critical flows: functional simulation runs 10-100x faster than cycle-accurate, enabling longer benchmark runs.

**When to use cycle-accurate simulation:**
- Microarchitecture evaluation: comparing pipeline depths, cache configurations, predictor designs.
- Performance measurement: counting cycles for a benchmark, computing CPI.
- RTL verification: checking that the implemented design matches the architectural specification.
- Power estimation: cycle-level activity factors are needed for power models.

**In practice:** Most RISC-V development starts with Spike (functional), moves to QEMU (faster functional emulation for OS work), and uses cycle-accurate RTL simulation only for specific performance or verification tasks.

---

### Question F2
**What is the RISC-V proxy kernel (pk) and what problem does it solve when running programs under Spike?**

**Answer:**

The proxy kernel (pk) is a minimal RISC-V supervisor-mode software layer that runs under Spike and forwards selected system calls from the RISC-V application to the host operating system.

**The problem it solves:** A bare-metal RISC-V binary has no OS beneath it. If the program calls `printf`, `malloc`, `open`, or any standard C library function that ultimately invokes an `ecall` (RISC-V system call instruction), there is no kernel to handle the ecall. The simulation would trap and halt.

pk sits in S-mode beneath the application (which runs in U-mode). When the application issues an ecall:
1. The trap is taken by pk (running in S-mode).
2. pk inspects the system call number and arguments.
3. pk forwards the call to the host OS using the HTIF (Host Target Interface) mechanism.
4. The host OS executes the call (e.g., calls the real `write()` function on the host).
5. The result is returned to the RISC-V application.

**What pk supports:**
- Standard POSIX syscalls: read, write, open, close, exit, brk (for malloc).
- Enough to run C programs that use `stdio.h`, `stdlib.h`, standard string functions.

**What pk does NOT support:**
- Multi-threading (`pthread_create` would not work).
- Networking, signals, or complex process management.
- Programs that require a real kernel.

For full Linux application support, QEMU user mode (`qemu-riscv64`) is the alternative: it emulates the CPU and intercepts syscalls, forwarding them to the host Linux kernel.

---

### Question F3
**How does QEMU user mode differ from QEMU system mode? Give a practical example of when each is the appropriate tool for RISC-V development.**

**Answer:**

**QEMU user mode (`qemu-riscv64`):**
- Emulates only the RISC-V CPU user-mode instruction set.
- Does not emulate hardware devices, interrupts, or a physical memory map.
- Intercepts `ecall` instructions and translates them to equivalent host system calls.
- Can run any RISC-V Linux ELF binary on an x86 host without installing a full RISC-V Linux system.

**QEMU system mode (`qemu-system-riscv64`):**
- Emulates a complete hardware platform: CPU, RAM, UART, interrupt controller (PLIC), timer (CLINT), and optionally virtio storage and network devices.
- The guest software interacts with emulated hardware, not the host hardware.
- A full bootloader (OpenSBI + U-Boot or GRUB) and Linux kernel run inside the emulator.
- The guest kernel runs in S-mode; M-mode is handled by OpenSBI.

**Practical examples:**

*Use QEMU user mode when:* You have cross-compiled a Linux application (`riscv64-unknown-linux-gnu-gcc`) and want to test it quickly on your x86 development machine without booting a VM or flashing hardware.
```bash
# Quick test of a cross-compiled benchmark
riscv64-unknown-linux-gnu-gcc -O2 -o coremark coremark.c
qemu-riscv64 -L /opt/riscv/sysroot ./coremark
# Output: CoreMark score printed to stdout
```

*Use QEMU system mode when:* You are developing or testing a Linux kernel driver for a RISC-V SoC, debugging U-Boot boot sequence, testing OpenSBI platform support, or verifying that a custom device driver correctly handles MMIO register interactions.
```bash
# Boot Linux to test a custom kernel module
qemu-system-riscv64 -machine virt -nographic -kernel Image \
  -append "console=ttyS0 root=/dev/vda" \
  -drive file=rootfs.qcow2,format=qcow2
```

---

## Tier 2 — Intermediate

### Question I1
**Describe the co-simulation methodology using Spike and an RTL simulator. What exactly is being compared at each step, and what kinds of bugs does this methodology catch that unit testing alone would miss?**

**Answer:**

**Co-simulation setup:**

The co-simulator runs Spike (the architectural golden reference) and the RTL simulation in lock-step. After each instruction retires in the RTL simulation, the co-simulator:

1. Reads the PC of the just-retired instruction and the values of all modified registers from the RTL simulation.
2. Steps Spike forward by one instruction from the same initial state.
3. Compares: PC, all 32 integer registers (and FP registers, CSRs if applicable).
4. If any value differs: record the mismatch with context (instruction opcode, input register values, expected vs. actual output) and halt simulation.

```
Co-sim check at each instruction:
  RTL retires instruction at PC=0x1234, writes x3=0xDEAD_BEEF
  Spike steps from same PC, computes x3=0xDEAD_BEEF  -> MATCH
  ...
  RTL retires instruction at PC=0x1280, writes x7=0x0000_0001
  Spike computes x7=0x0000_0002                       -> MISMATCH
  Log: "Divergence at PC=0x1280, instruction=0x003_3833 (ADD x7,x6,x3)
        RTL: x7=0x0000_0001; Spike: x7=0x0000_0002"
```

**Bugs caught by co-simulation that unit tests miss:**

1. **Corner-case ALU bugs:** A unit test for ADD might test `0 + 0`, `1 + 1`, and a few typical values. Co-simulation with random instruction streams exercises all 2^32 * 2^32 input combinations over millions of instructions, finding edge cases like carry propagation errors at specific bit boundaries.

2. **Pipeline interaction bugs:** A forwarding path bug may only manifest when a specific sequence of instructions (e.g., a load followed by a branch that reads the loaded value) executes. Unit tests for individual instructions never create this sequence. Random instruction streams hit it naturally.

3. **CSR interaction bugs:** Incorrect handling of status CSR bits (e.g., MSTATUS.MIE clearing incorrectly on trap entry) requires a trap to be triggered, a handler to run, and a return to U-mode — a sequence that only arises in integration testing.

4. **Exception handling correctness:** If an instruction at an odd address causes an instruction-misaligned exception, the co-simulator verifies that `mepc` and `mcause` are updated to exactly the correct values as specified by the ISA.

5. **Extension interaction bugs:** Enabling the F extension changes some integer instruction behaviors (in RV64, FCVT instructions must NaN-box). These interactions are tested correctly only when both F and I instructions execute in the same instruction stream.

---

### Question I2
**A developer runs a RISC-V program under Spike and sees `SIGILL` (illegal instruction). What are the three most likely causes, and how would you diagnose each using Spike's debugging capabilities?**

**Answer:**

**Cause 1: Missing ISA extension in Spike invocation**

The program uses an instruction from an extension (e.g., `MUL` from M, `FLD` from D, or a compressed instruction from C) but Spike was invoked without the corresponding `--isa` flag.

*Diagnosis:*
```bash
# Run with -l (log all instructions)
spike -l --isa=rv64imac pk ./program 2>trace.log
# Look at the last instruction before the fault in trace.log

# Also: check what ISA the binary was compiled for
riscv64-unknown-elf-readelf -A ./program | grep Tag_RISCV_arch
```

*Fix:*
```bash
# Ensure --isa matches the -march used during compilation
spike --isa=rv64imafd pk ./program
```

**Cause 2: Misaligned instruction fetch (RISC-V base ISA requires 4-byte alignment for RV32/RV64)**

A bug in control flow (corrupt stack, incorrect JALR target, corrupted return address) caused the PC to point to a non-word-aligned address. The CPU raises an instruction-address-misaligned exception.

*Diagnosis:*
```bash
spike -d pk ./program
# In the debugger, when the fault occurs:
(spike) reg 0 pc        # show current PC -- if odd or non-word-aligned, this is the cause
(spike) reg 0 ra        # check return address register for corruption
(spike) mem 0 <stack_ptr>  # inspect stack for corruption
```

**Cause 3: Executing data as instructions (PC corruption)**

A stack overflow, buffer overflow, or function pointer corruption caused the PC to point into a data section. The bytes at that address do not form a valid RISC-V instruction encoding.

*Diagnosis:*
```bash
# Use objdump to find what is at the faulting address
riscv64-unknown-elf-objdump -d ./program | grep -A5 "<faulting_addr>"

spike -d pk ./program
(spike) until pc 0 <last_good_pc>   # run until just before the fault
(spike) reg 0 sp                     # check stack pointer -- overflow?
(spike) reg 0 ra                     # check return address
(spike) mem 0 <sp>                   # inspect stack contents
```

The key debugging workflow is: (1) reproduce in Spike with `-l` to get an instruction trace, (2) identify the last valid instruction and the faulting PC, (3) work backward from the faulting PC to identify how control flow arrived there.

---

## Tier 3 — Advanced

### Question A1
**Explain the HTIF (Host Target Interface) mechanism in Spike in detail. How does a bare-metal RISC-V test signal "pass" or "fail" to Spike, and why is this relevant for the RISC-V architectural compliance test suite?**

**Answer:**

**HTIF mechanism:**

HTIF uses two memory-mapped registers at addresses determined by linker symbols in the ELF binary:

```
ELF symbols:
  .tohost   -- target writes here to signal the host
  .fromhost -- host writes here to send data to the target

Typical address (assigned by the linker script for compliance tests):
  tohost   @ 0x8000_1000 (example)
  fromhost @ 0x8000_1040 (example)
```

When the RISC-V program writes to `tohost`:
- Spike detects the write (by monitoring the memory address in the simulator).
- It interprets the value:
  - `tohost == 1`: exit with success (test PASS).
  - `tohost == (code << 1) | 1` with code != 0: exit with failure code.
  - High bits set: encoded system call request (read, write, open, etc.).

**Test pass/fail signalling in compliance tests:**
```assembly
# RISC-V compliance test "pass" idiom:
.section .tohost,"aw",@progbits
.align 3
.globl tohost
tohost: .dword 0

.section .fromhost,"aw",@progbits
.align 3
.globl fromhost
fromhost: .dword 0

# In the test:
    li  t0, 1
    la  t1, tohost
    sd  t0, 0(t1)      # write 1 to tohost -> Spike exits with success

# For a failure (non-zero test number):
    li  t0, (TEST_NUM << 1) | 1
    la  t1, tohost
    sd  t0, 0(t1)      # Spike exits with failure code TEST_NUM
```

**Why HTIF matters for RISC-V compliance testing:**

The RISC-V architectural compliance test suite (riscv-arch-test) uses exactly this mechanism to report results. Each test:
1. Executes a specific instruction or instruction sequence.
2. Compares the output with a pre-computed reference signature stored in memory.
3. Writes `1` to `tohost` if the signature matches (PASS) or a non-zero code if it does not (FAIL).

This makes HTIF the architectural "test infrastructure" that decouples the RISC-V test from the specific simulation environment. The same test binary can run on:
- Spike (reference simulation)
- QEMU
- RTL simulation (with an HTIF memory model)
- Physical hardware (with an HTIF memory-mapped register and a JTAG monitor)

The test does not need to know which environment it is in; it just writes to `tohost`.

---

### Question A2
**Describe how to add a custom RISC-V instruction to Spike using its plugin interface. What are the steps from instruction encoding to a simulated result, and what limitations does this approach have compared to co-simulation with RTL?**

**Answer:**

**Step 1: Choose the encoding**

Select an encoding in the custom-0 (opcode 0x0B), custom-1 (0x2B), custom-2 (0x5B), or custom-3 (0x7B) opcode spaces reserved by the RISC-V ISA for custom instructions:

```
Example: POPCOUNT rd, rs1 (count the number of set bits in rs1)
Encoding:
  [31:25] funct7 = 0b0000001
  [24:20] rs2    = 00000 (unused)
  [19:15] rs1    = source register
  [14:12] funct3 = 0b000
  [11:7]  rd     = destination register
  [6:0]   opcode = 0b0001011 (custom-0)
```

**Step 2: Implement the instruction semantic in C++**

Create a new .h file in Spike's `riscv/insns/` directory:

```cpp
// File: riscv/insns/popcount.h
// POPCOUNT rd, rs1 -- count population of set bits in rs1 (64-bit)

require_extension('X');   // mark as custom extension
int64_t  src = RS1;
int64_t  result = __builtin_popcountll((uint64_t)src);
WRITE_RD(result);
```

**Step 3: Register the instruction in the decode table**

In `riscv/encoding.h`, add the match and mask for the new instruction:
```c
#define MATCH_POPCOUNT 0x0200000B   // funct7=0x01, rs2=0, funct3=0, opcode=custom-0
#define MASK_POPCOUNT  0xFE00707F   // mask covers funct7, rs2, funct3, opcode
```

In `riscv/riscv.mk.in`, add `popcount` to the `insn_list`.

**Step 4: Build Spike with the new instruction**

```bash
cd riscv-isa-sim
./configure --prefix=/opt/spike
make -j$(nproc)
make install
```

**Step 5: Assemble and test using inline assembly**

```c
// Test program using the custom POPCOUNT instruction
#include <stdio.h>

static inline long riscv_popcount(long x) {
    long result;
    // Custom instruction encoding: .insn r OPCODE, funct3, funct7, rd, rs1, rs2
    asm volatile (
        ".insn r 0x0B, 0x0, 0x01, %0, %1, x0"
        : "=r"(result) : "r"(x)
    );
    return result;
}

int main() {
    printf("popcount(0xFF) = %ld\n", riscv_popcount(0xFF));  // should print 8
    return 0;
}
```

```bash
riscv64-unknown-elf-gcc -march=rv64imac -o test test.c
spike --isa=rv64imaXpopcount pk test
# Output: popcount(0xFF) = 8
```

**Limitations compared to RTL co-simulation:**

1. **No timing model:** Spike's custom instruction executes in zero simulated cycles. It cannot model a multi-cycle custom unit, pipeline stalls introduced by the custom instruction, or the impact on forwarding paths.

2. **No structural hazard modelling:** If the custom instruction uses a resource (e.g., a multiplier) that is already in use, Spike will not stall — it simply executes immediately.

3. **No microarchitectural state:** Spike cannot model the custom instruction's impact on cache state, branch prediction counters, or pipeline register pressure.

4. **Plugin isolation:** The Spike plugin runs trusted C++ code in the same process as the simulator. There is no isolation between the custom model and the rest of Spike — a bug in the custom code can corrupt Spike's internal state.

5. **Verification gap:** RTL co-simulation using Spike as a reference catches timing-independent bugs in the custom instruction's hardware implementation. Without RTL, only the functional correctness of the C++ model is verified, not the hardware.

For these reasons, Spike custom instruction simulation is appropriate for early software development and ISA exploration. For production validation, the custom instruction RTL must be co-simulated against the Spike functional model.
