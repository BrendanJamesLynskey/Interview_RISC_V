# Quiz: Ecosystem

## Instructions

15 multiple-choice questions covering the RISC-V toolchain (GCC/LLVM), simulation
(Spike and QEMU), the RISC-V debug specification, and custom instruction extensions.
Each question has exactly one correct answer. Work through all questions before
checking the answer key at the end.

Difficulty distribution: Questions 1-5 Fundamentals, Questions 6-11 Intermediate,
Questions 12-15 Advanced.

---

## Questions

### Q1 (Fundamentals)

Which compiler flag specifies the RISC-V ISA extensions that the GCC or Clang
compiler is allowed to emit instructions for?

- A) `-mabi`
- B) `-march`
- C) `-mtune`
- D) `-misa`

---

### Q2 (Fundamentals)

What does the `-mabi` flag control in the RISC-V GCC toolchain, and how does it
differ from `-march`?

- A) `-mabi` controls the linker script used; `-march` controls the assembler dialect.
- B) `-mabi` specifies the calling convention and floating-point register usage
     (e.g., `ilp32`, `lp64d`); `-march` specifies which ISA extensions may be used for
     code generation. Both must be consistent: you cannot specify `lp64d` ABI without
     at least `-march=rv64...d`.
- C) `-mabi` enables RISC-V privileged instructions; `-march` enables user-mode instructions.
- D) `-mabi` and `-march` are aliases for the same flag; either can be used interchangeably.

---

### Q3 (Fundamentals)

Spike is the official RISC-V ISA reference simulator. Which statement best describes
its primary role in the RISC-V ecosystem?

- A) Spike is a cycle-accurate RTL simulator used to measure exact hardware performance.
- B) Spike is a functional (instruction-set-level) simulator that models the RISC-V
     ISA precisely, serving as the golden reference for verifying software and hardware
     implementations.
- C) Spike is a full-system emulator that models peripheral devices, network interfaces,
     and DRAM controllers for embedded SoC development.
- D) Spike is a static analysis tool that checks RISC-V assembly code for correctness
     before loading it onto hardware.

---

### Q4 (Fundamentals)

The RISC-V debug specification defines a Debug Module (DM). What is the primary
function of the Debug Module?

- A) It is a CSR that the hart writes to when it encounters a breakpoint, recording the
     faulting instruction address.
- B) It is an external hardware block that a debugger host communicates with to gain
     control over one or more RISC-V harts — enabling halt, resume, single-step,
     register/memory access, and breakpoint management without modifying the hart's
     pipeline.
- C) It is a software daemon running on the RISC-V target that responds to GDB remote
     serial protocol packets over UART.
- D) It is the firmware component in M-mode responsible for forwarding debug
     exceptions to an external JTAG interface.

---

### Q5 (Fundamentals)

In the RISC-V specification, opcode space `custom-0` (opcode `0001011`) and
`custom-1` (opcode `0101011`) are reserved for what purpose?

- A) They are reserved for future standard extensions and must not be used by
     implementations.
- B) They are reserved for non-standard (custom) extensions; implementations may use
     these opcodes freely without conflicting with any present or future standard
     RISC-V extension.
- C) They encode 48-bit and 64-bit instruction prefixes respectively for variable-length
     standard extensions.
- D) They are the opcodes for CSR read and CSR write instructions.

---

### Q6 (Intermediate)

A developer builds a bare-metal RISC-V binary using:

```bash
riscv32-unknown-elf-gcc -march=rv32imc -mabi=ilp32 -O2 -o app.elf app.c
```

They then attempt to run it on hardware that only implements `rv32i` (no M or C
extension). Describe what will happen and why.

- A) The binary runs correctly; the `-march` flag only affects optimisation, not the
     actual instructions emitted.
- B) The binary will likely fault with an illegal instruction exception when the
     processor encounters a compressed (16-bit C-extension) or multiply/divide
     (M-extension) instruction that the hardware does not support.
- C) The binary runs correctly on `rv32i` hardware because GCC automatically detects
     missing hardware extensions at startup and emulates them in software.
- D) The linker will refuse to produce the binary if the target hardware does not
     support the specified `-march`, so the binary will never be created.

---

### Q7 (Intermediate)

The RISC-V debug specification defines two types of trigger: address/data match
triggers and instruction count triggers. A developer sets an address trigger on a
data address in TDATA1/TDATA2. What does this trigger cause when the hart executes a
load or store to that address?

- A) It writes the faulting address to `mtval` and generates an illegal instruction
     exception to M-mode.
- B) Depending on the trigger action configured in TDATA1, it either causes a debug
     mode entry (if `action=1`) so the debugger can inspect state, or generates a
     breakpoint exception (`cause=3`) to the trap handler (if `action=0`).
- C) It halts the entire SoC and signals a fault to the external JTAG controller
     unconditionally.
- D) It records the address in the performance counter `hpmcounter3` and continues
     execution without interruption.

---

### Q8 (Intermediate)

QEMU supports two modes of RISC-V simulation. What are they, and in what scenario
would you prefer each?

- A) Fast mode and accurate mode; fast mode skips instruction decoding, accurate mode
     performs full decode.
- B) User-mode emulation (`qemu-riscv32` / `qemu-riscv64`) and full-system emulation
     (`qemu-system-riscv32` / `qemu-system-riscv64`). User-mode emulates only the
     userspace ABI (translates syscalls to the host OS), suitable for running and
     testing Linux ELF binaries quickly. Full-system emulates the entire platform
     including UART, CLINT, PLIC, and boots a complete OS — suitable for OS development,
     firmware testing, and driver development.
- C) Interpreted mode and JIT mode; interpreted mode is used for ISA conformance
     testing, JIT mode for performance benchmarking.
- D) Single-hart mode and multi-hart mode; single-hart mode is used for embedded
     targets, multi-hart mode for server simulations.

---

### Q9 (Intermediate)

A custom RISC-V instruction must be encoded in the 32-bit instruction space. The
`custom-0` opcode is used. Which additional fields are typically available for
encoding the operation within a custom-0 instruction?

- A) Only the 7-bit opcode field; all other bits are reserved for future use.
- B) The full 25 bits above the opcode are available: the instruction can use `funct3`
     (3 bits), `funct7` (7 bits for R-type), and the register specifier fields, giving
     up to 10 bits of function discriminator alongside standard register operands — or
     alternatively, the upper 25 bits can encode a single large immediate.
- C) Only `funct3` (3 bits); the remaining bits are fixed to zero by the specification.
- D) Custom instructions must use the I-type format exclusively; R-type encoding is
     reserved for standard extensions.

---

### Q10 (Intermediate)

When adding a custom instruction to GCC, a developer writes a GCC intrinsic. Why are
intrinsics preferred over inline assembly for custom instructions in a larger codebase?

- A) Intrinsics run faster than inline assembly because the compiler omits the
     assembler step when intrinsics are used.
- B) Intrinsics are C/C++ function calls that the compiler knows map directly to
     specific instructions. Unlike inline assembly, they participate in register
     allocation and scheduling, allow the compiler to optimise surrounding code (e.g.,
     eliminate redundant moves), work across different compiler versions via a stable
     API, and are type-safe. Inline assembly inserts opaque text that the compiler
     cannot reason about.
- C) Intrinsics are required by the RISC-V ABI specification; inline assembly is not
     permitted in conforming code.
- D) Intrinsics automatically generate the corresponding LLVM IR, making them portable
     across all architectures, whereas inline assembly is architecture-specific.

---

### Q11 (Intermediate)

The `pk` (Proxy Kernel) is used alongside Spike. What does it provide, and when would
you use Spike without `pk`?

- A) `pk` is a GCC plug-in that generates Spike-compatible binary format; without it,
     Spike cannot load ELF files.
- B) `pk` is a minimal M-mode/S-mode kernel that handles system calls from user-space
     programs by proxying them to the host OS (e.g., `write()` becomes a host `write`).
     It allows running Linux userspace ELF binaries on Spike without a full OS. You
     would use Spike without `pk` when running bare-metal firmware, a full OS image, or
     when testing M-mode code directly — scenarios where the proxy syscall mechanism
     would be inappropriate or absent.
- C) `pk` provides the RISC-V privilege hardware model for Spike; without `pk`, Spike
     simulates only user-mode instructions.
- D) `pk` is the RISC-V compliance test suite runner for Spike; it validates the
     simulator against the ISA specification.

---

### Q12 (Advanced)

A team is adding a new custom accelerator instruction `CRC32.W rd, rs1` to a RISC-V
SoC. Describe the complete set of changes required to the GCC toolchain to emit this
instruction from C source code via an intrinsic, and ensure the linker and assembler
handle it correctly.

- A) Only the assembler needs to be modified; GCC automatically generates calls to
     any function named with the `__builtin_` prefix.
- B) The required changes are: (1) add the instruction encoding to the assembler
     (gas/binutils): define the mnemonic, operand syntax, and binary encoding in the
     RISC-V opcode table; (2) add the intrinsic definition to GCC: declare a new
     built-in function with the correct type signature and map it to an RTL pattern
     that emits the instruction; (3) add a machine description pattern (`.md` file
     entry in the GCC RISC-V backend) that describes the instruction's operands,
     constraints, and output template; (4) optionally add a C header (`<riscv_crc.h>`)
     wrapping the built-in with a user-friendly name. The linker requires no changes
     because the instruction is inline in the object file.
- C) Only the linker script needs updating to reserve opcode space; the compiler and
     assembler use this reservation to emit the instruction automatically.
- D) The RISC-V ISA requires that custom instructions be submitted to the RISC-V
     International Foundation for allocation before any toolchain changes can be made;
     no toolchain work is possible until the opcode is officially registered.

---

### Q13 (Advanced)

A developer uses Spike with the `--log-commits` flag and pipes the output to a file.
They observe that a specific instruction is executed 10 million times. They then run
the same binary under QEMU full-system emulation and the instruction count differs
significantly. What is the most likely explanation, and what does this reveal about
the two simulators?

- A) The difference is caused by QEMU's JIT recompiler skipping instructions for
     performance; QEMU is not a functionally correct RISC-V simulator.
- B) Spike is a functional ISA simulator that executes every architectural instruction
     exactly once per program flow, giving a precise instruction count. QEMU's TCG
     JIT recompiler translates blocks of guest instructions into host code; instruction
     counts reported by QEMU may differ from Spike because QEMU's performance counters
     measure translated block executions rather than individual guest instruction
     retirements, and the two simulators may handle interrupts, timer ticks, and
     peripheral interactions differently in full-system mode. For absolute instruction
     counts, Spike is the authoritative reference.
- C) The difference is caused by the fact that QEMU simulates a multi-hart system by
     default, so the instruction count is divided among harts; Spike always uses a
     single hart.
- D) Spike and QEMU use different instruction encodings internally; the count difference
     reflects decoding disagreements on certain compressed instructions.

---

### Q14 (Advanced)

The RISC-V debug specification defines the concept of "abstract commands." What are
abstract commands, and how do they allow a debugger to read/write registers and memory
without the processor executing a debug ROM?

- A) Abstract commands are CSR instructions embedded in a special debug trap handler
     that the DM injects into M-mode; the processor fetches and executes them as normal
     instructions.
- B) Abstract commands are operations the Debug Module can execute autonomously on
     behalf of the debugger host without requiring the hart to execute any instructions.
     The DM has dedicated hardware to perform register read/write and memory access
     operations by directly accessing the hart's register file and bus interface,
     bypassing the pipeline entirely. The debugger writes a command to the `command`
     CSR in the DM; the DM executes it and makes results available via `data0`/`data1`
     registers. This allows register and memory access even when the pipeline is stalled
     or corrupted.
- C) Abstract commands are RISC-V compressed (C-extension) instructions used only
     during debug mode, which execute in a dedicated abstract execution unit that is
     separate from the main pipeline.
- D) Abstract commands are the names for the JTAG TDI/TDO shift operations used to
     communicate between the host debugger and the on-chip debug transport module.

---

### Q15 (Advanced)

A RISC-V vendor wants to add a custom instruction that modifies both an integer
register and a CSR in a single operation (a combined read-modify-write of a hardware
counter and a result register). They use the `custom-3` opcode space. A colleague
warns that this design could cause problems with the RISC-V memory model and trap
handling. What specific architectural concern does the colleague have, and how should
it be addressed?

- A) The concern is that `custom-3` opcode space is allocated only for 64-bit
     instructions; a 32-bit encoding will be rejected by conforming assemblers.
- B) The concern is that an instruction with multiple architectural side effects
     (writing both a register and a CSR) creates difficulties for precise exception
     handling and speculative execution: if the instruction causes a trap partway
     through, the architectural state must be precisely defined (either both the
     register and CSR are updated, or neither). Out-of-order processors may begin
     executing the instruction and perform partial updates visible before commit. The
     solution is to treat the instruction as non-interruptible and non-speculative
     (similar to how AMOs are handled), define precise semantics for partial failure
     in the extension specification, and ensure that the microarchitecture writes both
     destinations atomically at commit — or separates the operation into two
     instructions with a defined ordering requirement.
- C) The concern is that the `custom-3` opcode space conflicts with the existing
     `FENCE` instruction encoding and the instruction will be misidentified by
     debuggers.
- D) The concern is that CSRs can only be accessed by CSR instructions (`CSRRW`,
     `CSRRS`, etc.) and a non-CSR instruction writing a CSR will silently be ignored
     by all conforming hardware implementations.

---

## Answer Key

| Q  | Answer | Difficulty    |
|----|--------|---------------|
| 1  | B      | Fundamentals  |
| 2  | B      | Fundamentals  |
| 3  | B      | Fundamentals  |
| 4  | B      | Fundamentals  |
| 5  | B      | Fundamentals  |
| 6  | B      | Intermediate  |
| 7  | B      | Intermediate  |
| 8  | B      | Intermediate  |
| 9  | B      | Intermediate  |
| 10 | B      | Intermediate  |
| 11 | B      | Intermediate  |
| 12 | B      | Advanced      |
| 13 | B      | Advanced      |
| 14 | B      | Advanced      |
| 15 | B      | Advanced      |

---

## Detailed Explanations

### Q1 - Answer: B (`-march`)

`-march` (machine architecture) tells GCC and Clang exactly which ISA extensions are
available on the target processor and thus which instructions the compiler is permitted
to emit. For example, `-march=rv32imc` enables the base integer ISA plus the M
(multiply/divide) and C (compressed) extensions.

- **A (`-mabi`)** specifies the calling convention and floating-point ABI (e.g.,
  `ilp32`, `lp64d`), not the instruction set.
- **C (`-mtune`)** hints to the compiler about which processor microarchitecture to
  optimise instruction scheduling for (e.g., latencies, throughput), but does not
  change which instructions can be emitted.
- **D (`-misa`)** is not a valid GCC flag.

---

### Q2 - Answer: B

`-march` and `-mabi` serve complementary but distinct roles:

- `-march=rv32imc` permits the compiler to generate I, M, and C instructions. It says
  nothing about how function arguments, return values, and floating-point values are
  passed.
- `-mabi=ilp32` specifies integer calling convention (int/long/pointer = 32 bits) and
  that no hardware floating-point registers are used for argument passing. `-mabi=ilp32f`
  would use `f0`-`f7` for float arguments; `-mabi=ilp32d` would use them for double.

Mismatching them causes link errors (e.g., a library compiled with `-mabi=ilp32d` and
application code compiled with `-mabi=ilp32` will disagree on where float arguments
are placed).

- **A** is wrong: neither flag controls the linker script directly.
- **C** is wrong: both flags apply equally to user-mode and privileged code.
- **D** is wrong: they are not aliases; using them interchangeably will produce
  incorrect binaries or build errors.

---

### Q3 - Answer: B (functional ISA reference simulator)

Spike (named after the RISC-V golden spike metaphor) is a software model of the
RISC-V ISA that executes every instruction exactly as the specification defines. It is
not cycle-accurate and does not model pipeline stages, cache miss latencies, or branch
prediction. Its value is correctness: any behaviour it exhibits for a given program
input should be reproducible on any conforming hardware implementation. Developers use
Spike to:
- Verify that software runs correctly before hardware is available.
- Cross-check hardware RTL simulators (if the RTL diverges from Spike, the RTL has a
  bug).
- Develop and test new ISA extensions before adding them to hardware.

- **A** is wrong: cycle-accurate simulation requires RTL simulators like VCS, Questa,
  or Verilator running the actual processor RTL.
- **C** is wrong: Spike has minimal peripheral modelling; QEMU is the better choice
  for full-system SoC emulation.
- **D** is wrong: Spike executes the binary dynamically; it is not a static analysis tool.

---

### Q4 - Answer: B (external hardware control block)

The Debug Module is a hardware block, distinct from the hart(s) it controls, that
provides the debugger host (e.g., OpenOCD + GDB) with a standardised interface to:
- Halt and resume individual harts.
- Single-step a hart one instruction at a time.
- Read and write general-purpose registers and CSRs (via abstract commands).
- Read and write physical memory.
- Set hardware breakpoints and watchpoints using the trigger module.

The DM is accessed via a Debug Transport Module (DTM) — typically JTAG — which
implements the physical connection from the debug host to the on-chip DM.

- **A** is wrong: `mtval` is a standard trap CSR; it is not the DM.
- **C** describes `gdbserver`, which is a software component used for Linux userspace
  debugging over a network or serial port, not the hardware DM.
- **D** is wrong: the DM is hardware, not M-mode firmware.

---

### Q5 - Answer: B (reserved for non-standard/custom extensions)

The RISC-V base ISA specification permanently reserves four opcode regions for custom
use: `custom-0` (`0001011`), `custom-1` (`0101011`), `custom-2` (`1011011`), and
`custom-3` (`1111011`). The specification guarantees that standard RISC-V extensions
will never claim these opcodes, so SoC vendors and research groups can add proprietary
instructions without risking future conflicts when the standard ISA evolves.

- **A** is wrong: these opcodes are explicitly designated for custom use, not for
  future standard extensions.
- **C** is wrong: variable-length instruction encodings use the lowest bits of the
  instruction word, not these specific opcodes.
- **D** is wrong: CSR instructions use opcode `1110011` (SYSTEM group).

---

### Q6 - Answer: B (illegal instruction fault on hardware)

When GCC is given `-march=rv32imc`, it is permitted to emit:
- Compressed 16-bit instructions (C extension): e.g., `C.ADDI`, `C.LW`
- Multiply/divide instructions (M extension): e.g., `MUL`, `DIV`

An `rv32i`-only processor has no hardware decoders for these instructions. When the
hardware encounters a 16-bit instruction encoding (bits [1:0] != `11`) or an M-extension
opcode, it raises an illegal instruction exception. The binary will crash at the first
such instruction.

The correct fix is to compile with `-march=rv32i` to ensure only base integer
instructions are emitted, or to enable software emulation traps in the M-mode firmware
(which intercepts the illegal instruction exception and emulates the missing instruction
in software — a valid but slow approach for embedded systems).

- **A** is wrong: `-march` directly controls which instruction encodings are emitted.
- **C** is wrong: GCC does not add automatic runtime extension detection; the binary is
  compiled once for a fixed target.
- **D** is wrong: the linker operates on ELF files and does not check hardware capability;
  it does not refuse to link based on `-march`.

---

### Q7 - Answer: B (debug mode entry or breakpoint exception)

The RISC-V trigger specification (part of the debug spec) defines triggers as hardware
comparators that fire when a specific condition is met (e.g., a load/store address
matching TDATA2). The action taken on trigger fire is controlled by the `action` field
in TDATA1:

- `action = 0`: Generate a breakpoint exception (trap cause 3). The hart takes a
  normal trap to M-mode or S-mode. This is useful for software debuggers (e.g., gdb
  configured to use hardware watchpoints via the OS kernel's `ptrace` interface).
- `action = 1`: Enter debug mode. The hart halts and the Debug Module takes control.
  The debugger host can then inspect state before resuming.

- **A** is wrong: `mtval` is populated on page faults and other specific exceptions,
  not by the trigger hardware directly; and the exception type is a breakpoint, not an
  illegal instruction.
- **C** is wrong: the action is configurable; it does not unconditionally halt the SoC.
- **D** is wrong: performance counters are not connected to the trigger action mechanism.

---

### Q8 - Answer: B (user-mode vs. full-system)

QEMU provides two distinct simulation modes for RISC-V:

**User-mode emulation** (`qemu-riscv32`, `qemu-riscv64`): Translates RISC-V
instructions into host instructions using TCG (Tiny Code Generator) JIT. Linux system
calls in the guest binary are intercepted and forwarded to the host OS. This is very
fast and requires no guest OS. Use cases: running RISC-V Linux ELF binaries on an
x86 development machine, running RISC-V test suites, CI for cross-compiled software.

**Full-system emulation** (`qemu-system-riscv32`, `qemu-system-riscv64`): Emulates
the complete hardware platform including the processor, DRAM, UART, CLINT (Core-Local
Interruptor), PLIC (Platform-Level Interrupt Controller), VirtIO devices, and more.
Boots a complete firmware + OS stack. Use cases: OS kernel development and debugging,
device driver development, firmware (OpenSBI/U-Boot) testing, SMP Linux boot testing.

- **A** is wrong: there is no documented "fast mode" and "accurate mode" in QEMU's
  RISC-V support as top-level modes.
- **C** is wrong: while QEMU does use a JIT internally (TCG), this is an implementation
  detail, not a user-facing mode selection.
- **D** is wrong: both user-mode and full-system QEMU support multi-hart; this is not
  the distinction between modes.

---

### Q9 - Answer: B (25 bits available above opcode)

The 7-bit opcode occupies bits [6:0]. The remaining 25 bits ([31:7]) are available to
the custom extension designer. Common approaches include:

- **R-type layout**: Use the standard `rd` [11:7], `funct3` [14:12], `rs1` [19:15],
  `rs2` [24:20], `funct7` [31:25] fields. This provides `funct3` (3 bits) + `funct7`
  (7 bits) = 10 bits of opcode disambiguation, allowing 1024 distinct operations using
  two register sources and one destination — all consistent with existing register file
  read/write ports.
- **I-type with immediate**: Use `rd`, `rs1`, and a 12-bit immediate in the upper bits,
  sacrificing opcode diversity for a larger immediate operand.
- **Large immediate**: Abandon standard register fields entirely and use all 25 bits as
  a single immediate field (similar to U-type encoding).

The key point is that the RISC-V specification imposes no constraints on bits above the
opcode in custom opcode space.

- **A** is wrong: only the 7-bit opcode is architecturally fixed; all 25 remaining bits
  are free for custom use.
- **C** is wrong: the specification does not restrict custom instructions to only using
  `funct3`.
- **D** is wrong: no such constraint on instruction format exists in custom opcode space.

---

### Q10 - Answer: B (type safety, register allocation, compiler reasoning)

GCC built-in intrinsics (declared with `__builtin_riscv_crc32_w(...)` or similar) are
integrated into the compiler's internal representation. This means:

- The compiler performs register allocation: it assigns the correct registers to the
  intrinsic's inputs and outputs and can eliminate redundant moves.
- The compiler can schedule the intrinsic relative to surrounding instructions based
  on latency and throughput information provided in the machine description.
- Type checking at compile time catches incorrect argument types.
- The intrinsic API is stable even if the underlying machine description changes.

Inline assembly uses `asm volatile("crc32.w %0, %1" : "=r"(out) : "r"(in))` which is
opaque to the compiler's optimiser: it cannot move, combine, or eliminate the assembly
block (if `volatile` is specified), and the `clobber` list must be manually maintained.

- **A** is wrong: intrinsics and inline assembly both go through the assembler; the
  performance difference is in optimisation quality, not the compilation pipeline.
- **C** is wrong: the RISC-V ABI does not mandate use of intrinsics.
- **D** is wrong: intrinsics are architecture-specific; they do not produce portable
  LLVM IR that runs on other architectures.

---

### Q11 - Answer: B (`pk` proxies syscalls; omit it for bare-metal and OS work)

The Proxy Kernel (`pk`) is a minimal supervisor-mode stub that runs on Spike and
implements a Linux-like syscall interface by forwarding calls (via HTIF — Host-Target
Interface) to the simulation host. When you run `spike pk my_app`, the binary thinks it
is running under Linux: `write()` prints to the host terminal, `read()` reads from the
host stdin, `exit()` terminates the simulation.

You would omit `pk` when:
- Running a complete OS image: `spike -m256 bbl linux.bin` boots OpenSBI + Linux.
- Testing bare-metal firmware that sets up its own M-mode trap handlers.
- Verifying M-mode CSR behaviour directly without OS interference.
- Running RISC-V compliance tests that target specific privilege-level behaviour.

- **A** is wrong: Spike can load standard ELF files directly without `pk`.
- **C** is wrong: `pk` provides syscall emulation, not privilege hardware modelling;
  Spike models all privilege levels regardless of whether `pk` is loaded.
- **D** is wrong: the RISC-V compliance test suite is a separate project (`riscv-tests`
  or `riscv-arch-test`); `pk` is not its runner.

---

### Q12 - Answer: B (assembler + GCC backend + machine description + header)

Adding a custom instruction to GCC requires changes at multiple levels of the
compilation stack:

1. **GNU Assembler (gas, part of binutils)**: The `riscv-opc.c` and `riscv-opc.h`
   files in binutils define the RISC-V opcode table. Add an entry with the mnemonic
   (`crc32.w`), operand encoding string, instruction mask, and match value. After this,
   the assembler can assemble hand-written `.S` files using the new instruction.

2. **GCC machine description** (`gcc/config/riscv/riscv.md`): Add a `define_insn`
   pattern that describes the instruction's inputs (register constraints), outputs,
   side effects, and the assembly template string (`"crc32.w\t%0,%1"`). This links
   the RTL (Register Transfer Language) representation inside GCC to the instruction.

3. **GCC built-in declaration** (`gcc/config/riscv/riscv-builtins.cc` and
   `riscv-ftypes.def`): Register the built-in function `__builtin_riscv_crc32_w` with
   its type signature (`unsigned int (unsigned int)`) and map it to the RTL pattern.

4. **User-facing header** (optional): Ship `<riscv_crc.h>` that provides
   `static inline uint32_t crc32_w(uint32_t x) { return __builtin_riscv_crc32_w(x); }`
   for user convenience.

The linker does not need changes because the instruction is emitted inline and there
is no separate library to link.

- **A** is wrong: GCC does not auto-generate instruction emission for `__builtin_`
  functions; each must be explicitly defined in the machine description.
- **C** is wrong: the linker script allocates memory sections, not opcode space.
- **D** is wrong: nothing prevents adding custom toolchain support before submitting
  to RISC-V International; the `custom-0` to `custom-3` opcode spaces exist precisely
  to allow vendor-specific extensions without any approval process.

---

### Q13 - Answer: B (Spike is the reference; QEMU counts differ for architectural reasons)

Spike executes each RISC-V instruction one at a time in strict program order and
`--log-commits` records each instruction retirement. The count is architecturally
precise.

QEMU's TCG JIT compiles blocks of guest instructions into host code. Its internal
instruction counting (`-icount`) approximates guest instruction counts but is subject
to block chaining, partial block execution, and differences in how timer interrupts
are injected. In full-system mode, timer-driven preemption and interrupt handling
introduce context switches that do not occur in the same sequence on Spike. The result
is a legitimately different execution path — not a QEMU bug, but a consequence of
running a full OS with timers, interrupts, and scheduling.

- **A** is wrong: QEMU is functionally correct for the instructions it models; the
  count difference is a consequence of different simulation architecture, not errors.
- **C** is wrong: both simulators default to single-hart unless explicitly configured
  otherwise, and single-hart configuration is easily verified.
- **D** is wrong: both simulators implement the same RISC-V encoding; encoding
  disagreements would be bugs (known bugs are rare and well-documented).

---

### Q14 - Answer: B (DM hardware executes commands autonomously)

Abstract commands are the mechanism by which a debugger (via the DM) can access
registers and memory without injecting instructions into the hart's instruction stream.
The DM has a hardware state machine that, upon receiving an abstract command from the
host, directly reads or writes the requested resource:

- **Access Register command**: The DM directly reads or writes a GPR or CSR using
  hardware that connects to the register file ports, bypassing the hart's pipeline.
  The result is placed in DM data registers (`data0`, `data1`) for the host to read.
- **Quick Access command**: The DM halts the hart, executes a sequence from the
  Program Buffer (a small RAM in the DM), then resumes — allowing more complex operations.

The distinction from simple pipeline injection: abstract register access does not
require the hart to execute any instruction. This is critical when the pipeline is
stalled or the hart is in an error state.

- **A** is wrong: abstract commands that use the Program Buffer do inject instructions,
  but abstract register access commands do not; describing all abstract commands as
  "trap handlers injected into M-mode" is incorrect.
- **C** is wrong: abstract commands have nothing to do with the C extension.
- **D** is wrong: JTAG shift operations are the transport mechanism (part of the DTM),
  not abstract commands.

---

### Q15 - Answer: B (precise exception semantics for multi-write instructions)

RISC-V's trap handling requires precise exceptions: when an instruction causes a fault,
the architectural state is exactly as it would be if all prior instructions had
completed and the faulting instruction had not executed at all. An instruction that
writes both a GPR and a CSR atomically complicates this in two ways:

1. **Partial update on fault**: If the instruction itself can cause a fault (e.g., if
   it also performs a memory access), must both writes have occurred, or neither? The
   spec must define this unambiguously.

2. **Out-of-order execution**: In an OoO processor, an instruction may compute its
   results speculatively and write to physical (renamed) registers before commit. A
   CSR write is typically not renamed — it is a global side effect. If the instruction
   writes the CSR before commit and the instruction is later squashed (e.g., due to a
   preceding branch misprediction), the CSR modification must be undone. Most OoO
   implementations do not support CSR write rollback.

The practical solution is to define the instruction as serialising (like `FENCE` or
`CSRRW`): it drains the pipeline before executing and commits atomically. This
guarantees precise semantics at the cost of pipeline disruption for every execution.
Alternatively, separating the operation into a standard CSR instruction plus the custom
register-writing instruction with a defined ordering removes the dual-write problem.

- **A** is wrong: `custom-3` (opcode `1111011`) is a 32-bit opcode space (bits [1:0]
  = `11`), not a 64-bit encoding.
- **C** is wrong: `custom-3` is `1111011`; `FENCE` is `0001111`. They do not overlap.
- **D** is wrong: RISC-V does not architecturally prohibit non-CSR instructions from
  modifying CSR-like state in custom hardware; the hardware can be designed however
  the implementer chooses within the custom opcode space.
