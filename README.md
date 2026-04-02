# RISC-V ISA Interview Preparation

[![RISC-V](https://img.shields.io/badge/Architecture-RISC--V-blue.svg)](https://riscv.org/)

This repository contains comprehensive interview preparation materials for RISC-V instruction set architecture, covering the ISA fundamentals, standard extensions, privilege modes, microarchitecture concepts, and ecosystem tools.

## Table of Contents

- [01 ISA Fundamentals](#01-isa-fundamentals)
- [02 Extensions](#02-extensions)
- [03 Privilege Architecture](#03-privilege-architecture)
- [04 Microarchitecture](#04-microarchitecture)
- [05 Ecosystem](#05-ecosystem)
- [06 Quizzes](#06-quizzes)
- [How to Use](#how-to-use)
- [Contributing](#contributing)
- [Related Repositories](#related-repositories)

## 01 ISA Fundamentals

Core concepts of the RISC-V instruction set architecture.

- [Instruction Formats](01_isa_fundamentals/instruction_formats.md)
- [RV32I Base Instructions](01_isa_fundamentals/rv32i_base_instructions.md)
- [Immediate Encoding](01_isa_fundamentals/immediate_encoding.md)
- [ABI and Calling Convention](01_isa_fundamentals/abi_and_calling_convention.md)
- [Worked Problems](01_isa_fundamentals/worked_problems/)
  - [Problem 01: Instruction Decoding](01_isa_fundamentals/worked_problems/problem_01_instruction_decoding.md)
  - [Problem 02: Branch Offset](01_isa_fundamentals/worked_problems/problem_02_branch_offset.md)
  - [Problem 03: Function Call Stack](01_isa_fundamentals/worked_problems/problem_03_function_call_stack.md)

## 02 Extensions

Standard RISC-V extensions and their applications.

- [M Extension: Multiply/Divide](02_extensions/m_extension_multiply_divide.md)
- [C Extension: Compressed](02_extensions/c_extension_compressed.md)
- [F/D Extensions: Floating Point](02_extensions/f_d_extensions_floating_point.md)
- [A Extension: Atomics](02_extensions/a_extension_atomics.md)
- [B Extension: Bitmanip](02_extensions/b_extension_bitmanip.md)
- [V Extension: Vector](02_extensions/v_extension_vector.md)
- [Worked Problems](02_extensions/worked_problems/)
  - [Problem 01: Compressed Encoding](02_extensions/worked_problems/problem_01_compressed_encoding.md)
  - [Problem 02: Atomic Sequence](02_extensions/worked_problems/problem_02_atomic_sequence.md)
  - [Problem 03: Vector Operation](02_extensions/worked_problems/problem_03_vector_operation.md)

## 03 Privilege Architecture

Privilege modes, memory management, and protection mechanisms.

- [Privilege Modes: M, S, U](03_privilege_architecture/privilege_modes_m_s_u.md)
- [CSR Registers](03_privilege_architecture/csr_registers.md)
- [Trap Handling](03_privilege_architecture/trap_handling.md)
- [Virtual Memory: SV32/SV39](03_privilege_architecture/virtual_memory_sv32_sv39.md)
- [PMP: Physical Memory Protection](03_privilege_architecture/pmp_physical_memory_protection.md)
- [Worked Problems](03_privilege_architecture/worked_problems/)
  - [Problem 01: Trap Delegation](03_privilege_architecture/worked_problems/problem_01_trap_delegation.md)
  - [Problem 02: Page Table Walk](03_privilege_architecture/worked_problems/problem_02_page_table_walk.md)
  - [Problem 03: PMP Configuration](03_privilege_architecture/worked_problems/problem_03_pmp_configuration.md)

## 04 Microarchitecture

Implementation-level concepts for RISC-V processors.

- [Pipeline Design](04_microarchitecture/pipeline_design.md)
- [Hazards and Forwarding](04_microarchitecture/hazards_and_forwarding.md)
- [Branch Prediction](04_microarchitecture/branch_prediction.md)
- [Out-of-Order Basics](04_microarchitecture/out_of_order_basics.md)
- [Worked Problems](04_microarchitecture/worked_problems/)
  - [Problem 01: Pipeline Hazard](04_microarchitecture/worked_problems/problem_01_pipeline_hazard.md)
  - [Problem 02: Forwarding Logic](04_microarchitecture/worked_problems/problem_02_forwarding_logic.md)
  - [Problem 03: Branch Penalty](04_microarchitecture/worked_problems/problem_03_branch_penalty.md)

## 05 Ecosystem

Toolchain, simulators, debugging, and custom extensions.

- [Toolchain: GCC/LLVM](05_ecosystem/toolchain_gcc_llvm.md)
- [Simulation: Spike and QEMU](05_ecosystem/simulation_spike_qemu.md)
- [Debug Specification](05_ecosystem/debug_specification.md)
- [Custom Extensions](05_ecosystem/custom_extensions.md)
- [Worked Problems](05_ecosystem/worked_problems/)
  - [Problem 01: Assembly Debugging](05_ecosystem/worked_problems/problem_01_assembly_debugging.md)
  - [Problem 02: Custom Instruction](05_ecosystem/worked_problems/problem_02_custom_instruction.md)
  - [Problem 03: Compliance Testing](05_ecosystem/worked_problems/problem_03_compliance_testing.md)

## 06 Quizzes

Self-assessment quizzes covering each major topic area.

- [Quiz: ISA Fundamentals](06_quizzes/quiz_isa.md)
- [Quiz: Extensions](06_quizzes/quiz_extensions.md)
- [Quiz: Privilege Architecture](06_quizzes/quiz_privilege.md)
- [Quiz: Microarchitecture](06_quizzes/quiz_microarchitecture.md)
- [Quiz: Ecosystem](06_quizzes/quiz_ecosystem.md)

## How to Use

This repository is structured for progressive learning and interview preparation:

1. **Start with fundamentals**: Begin with 01_isa_fundamentals to understand the RISC-V instruction format, base instruction set, and calling conventions.

2. **Learn extensions**: Progress through 02_extensions to understand how the ISA expands with multiply/divide, compressed instructions, floating-point, atomics, and vector operations.

3. **Study privilege modes**: Review 03_privilege_architecture to understand how operating systems interact with RISC-V at the privilege level, including memory management and protection.

4. **Explore microarchitecture**: Read 04_microarchitecture to understand how RISC-V processors are built, including pipeline design, hazards, and branch prediction.

5. **Understand the ecosystem**: Study 05_ecosystem to learn about toolchains, simulators, debugging, and how to extend RISC-V with custom instructions.

6. **Test your knowledge**: Use the quizzes in 06_quizzes to assess your understanding and identify areas for deeper study.

Each section includes worked problems that demonstrate practical application of the concepts. Use these to verify your understanding before moving forward.

## Contributing

Contributions are welcome. Please follow these guidelines:

- Maintain clarity and technical accuracy
- Follow the existing markdown structure
- Add examples and worked problems where appropriate
- Ensure all links are relative and functional
- Test any code examples or technical claims

For significant additions, please open an issue first to discuss the proposed changes.

## Related Repositories

These repositories complement this interview preparation material:

- [RISCV_RV32I_SingleCycle](https://github.com/BrendanJamesLynskey/RISCV_RV32I_SingleCycle) - Single-cycle RISC-V processor implementation
- [RISCV_RV32IMC_5stage](https://github.com/BrendanJamesLynskey/RISCV_RV32IMC_5stage) - 5-stage pipelined RISC-V processor with M and C extensions
- [RISCV_SoC](https://github.com/BrendanJamesLynskey/RISCV_SoC) - Complete RISC-V System-on-Chip design
- [RISCV_IOMMU](https://github.com/BrendanJamesLynskey/RISCV_IOMMU) - RISC-V IOMMU implementation
- [RISCV_DMA](https://github.com/BrendanJamesLynskey/RISCV_DMA) - RISC-V DMA controller design
