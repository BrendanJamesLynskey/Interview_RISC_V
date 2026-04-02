# Pipeline Design

## Prerequisites
- RISC-V RV32I instruction set and encoding formats
- Basic digital logic: combinational vs. sequential circuits, flip-flops
- Computer arithmetic: integer addition, comparison, shift operations

---

## Concept Reference

### Single-Cycle vs. Pipelined Execution

A **single-cycle processor** completes every instruction in one clock cycle. The clock period must be long enough to accommodate the slowest instruction's total combinational path — typically a load word (LW), which traverses the instruction memory, register file read, ALU, data memory, and register file write in one cycle.

```
Single-cycle critical path (LW instruction):
  PC register
    -> Instruction memory     ~200 ps
    -> Register file read     ~150 ps
    -> ALU (add base + imm)   ~180 ps
    -> Data memory read       ~250 ps
    -> Register file write    ~100 ps
  Total combinational delay:   ~880 ps
  Required clock period:       >= 880 ps => ~1.14 GHz maximum

  All other instructions are penalised by LW's slow data memory access,
  even though they do not use data memory at all.
```

A **pipelined processor** divides the datapath into stages separated by pipeline registers. Each stage completes its work in one cycle; multiple instructions occupy different stages simultaneously.

```
5-stage pipeline — instructions in flight:

Cycle:    1    2    3    4    5    6    7    8    9
I1:      IF   ID   EX   MEM  WB
I2:           IF   ID   EX   MEM  WB
I3:                IF   ID   EX   MEM  WB
I4:                     IF   ID   EX   MEM  WB
I5:                          IF   ID   EX   MEM  WB

  After the pipeline fills, one instruction completes every cycle (CPI = 1 ideally).
  Single-cycle: one instruction completes every 880 ps.
  Pipelined:    each stage runs at ~200 ps (the max stage latency).
                Throughput = 1 instruction / 200 ps = 5 GHz ideally.
```

### The Classic 5-Stage RISC-V Pipeline

```
IF  — Instruction Fetch
       PC -> instruction memory -> 32-bit instruction word
       PC+4 is computed for sequential fetch
       Branch target may be computed here (or in ID/EX depending on design)

ID  — Instruction Decode / Register File Read
       Decode opcode, funct3, funct7
       Read rs1, rs2 from register file
       Sign-extend immediate (I-, S-, B-, U-, J-type)
       Generate control signals: RegWrite, MemRead, MemWrite, Branch, ALUSrc, ...

EX  — Execute
       ALU computes: arithmetic, logical, shift, comparison, address
       ALU operands: rs1 vs. (rs2 or sign-extended immediate)
       Branch condition evaluated; branch target address = PC + imm

MEM — Memory Access
       Load: send ALU-computed address to data memory; receive read data
       Store: send ALU-computed address and rs2 value to data memory
       Non-memory instructions: pass ALU result through unchanged

WB  — Write Back
       Write result to rd in register file
       Result source: ALU output (R-type, I-type ALU) or memory read data (load)
```

### Pipeline Registers

Each stage boundary holds all values needed by downstream stages. The register names reflect the stages they separate:

```
IF/ID register  — holds: PC+4, instruction[31:0]

ID/EX register  — holds: PC+4, rs1_data, rs2_data, sign_ext_imm,
                          rd (destination register index),
                          control signals: RegWrite, MemRead, MemWrite,
                          Branch, ALUSrc, ALUOp[1:0], MemToReg

EX/MEM register — holds: branch_target_PC, zero_flag (branch condition),
                          ALU_result, rs2_data (for store), rd,
                          control signals: RegWrite, MemRead, MemWrite,
                          Branch, MemToReg

MEM/WB register — holds: ALU_result, mem_read_data, rd,
                          control signals: RegWrite, MemToReg
```

The critical design principle: **only propagate signals that are still needed**. The destination register index `rd` must flow all the way to WB so that the write-back stage knows where to write. Control signals are consumed and dropped at their respective stages.

### Stage Balancing and the Critical Path

Throughput is limited by the slowest pipeline stage. Good design aims to balance stage latencies:

```
Unbalanced example:
  IF:  200 ps
  ID:  100 ps
  EX:  350 ps   <- bottleneck
  MEM: 300 ps
  WB:  100 ps
  Clock period must be >= 350 ps; IF, ID, WB stages have slack.

Balanced design goal:
  All stages ~200 ps => clock period = 200 ps => 5 GHz (theoretical).
  In practice: register setup/hold time, routing overhead add ~20-30 ps per stage.
```

Deep pipelines (10-20 stages, as in Intel Sandy Bridge or ARM Cortex-A57) balance stages more finely to achieve higher clock frequencies, but increase branch misprediction penalties.

### CPI Analysis

Cycles Per Instruction is the key performance metric:

```
CPI = CPI_ideal + CPI_stalls

CPI_ideal = 1 (one instruction completes per cycle once pipeline is full)

Sources of stall cycles:
  Load-use hazard:         1 stall cycle per occurrence
  Branch (no prediction):  1-3 stall cycles (depends on when branch is resolved)
  Cache miss (I$):         typically 10-200 stall cycles
  Cache miss (D$):         typically 10-200 stall cycles per load/store miss

Example workload:
  30% loads, 20% branches, 5% loads followed immediately by a dependent instruction
  Branch resolved in EX stage (2-cycle penalty if taken, 0 if not taken)
  Assume 15% branch taken rate, no data cache misses.

  Stall contributions:
    Load-use stalls: 0.05 instructions/instr * 1 cycle = 0.05 cycles/instr
    Branch stalls:   0.20 * 0.15 * 2 cycles = 0.06 cycles/instr
    (assuming predict-not-taken, penalty only on taken branches)

  CPI = 1.00 + 0.05 + 0.06 = 1.11
```

### RISC-V Pipeline Control Signal Summary

```
Instruction  RegWrite  ALUSrc  MemRead  MemWrite  Branch  MemToReg  ALUOp
-----------  --------  ------  -------  --------  ------  --------  -----
R-type          1        0       0        0         0       0        10
I-type ALU      1        1       0        0         0       0        10
LW              1        1       1        0         0       1        00
SW              0        1       0        1         0       x        00
BEQ             0        0       0        0         1       x        01
JAL             1        x       0        0         0       0        11
LUI             1        1       0        0         0       0        11

ALUOp encoding: 00=add, 01=subtract/compare, 10=R-type (use funct3/funct7), 11=special
```

---

## Tier 1 — Fundamentals

### Question F1
**What are the five stages of the classic RISC-V pipeline? What does each stage do, and what hardware resources does it use?**

**Answer:**

The five stages and their hardware resources:

| Stage | Name              | Operations performed                                        | Hardware used                        |
|-------|-------------------|-------------------------------------------------------------|--------------------------------------|
| IF    | Instruction Fetch | Read instruction at current PC; compute PC+4               | PC register, instruction memory/cache|
| ID    | Decode / Reg Read | Decode opcode; read rs1, rs2; sign-extend immediate        | Decoder logic, register file (2 read ports)|
| EX    | Execute           | ALU operation; compute branch target; evaluate branch      | ALU, adder (branch target), mux      |
| MEM   | Memory Access     | Load from / store to data memory                           | Data memory/cache                    |
| WB    | Write Back        | Write result to rd in register file                        | Register file (1 write port)         |

**Why this division:** The five stages map to the five natural phases of instruction execution. Each boundary corresponds to a structural resource boundary (memory access, register file access, ALU computation), making stage latencies roughly equal without excessive logic splitting.

**Common mistake:** Candidates forget that ID and WB both access the register file. This creates a structural hazard if both stages simultaneously need the register file — solved by writing in the first half of the WB cycle and reading in the second half (register file internal forwarding), or by simply forwarding the WB result as a special forwarding case.

---

### Question F2
**What is a pipeline register and why is it necessary? Name all five pipeline registers in a 5-stage design and describe what each holds.**

**Answer:**

A pipeline register is a bank of flip-flops inserted between consecutive pipeline stages. It is necessary because:
1. It captures the outputs of one stage so the next stage can read stable values on the following clock edge.
2. It allows two stages to work simultaneously on different instructions without one stage's combinational logic overwriting values the previous stage still needs.

Without pipeline registers, the combinational paths of all stages would be connected directly, forcing the entire instruction through all five stages in a single (very long) clock cycle — which is just a single-cycle processor.

The five pipeline registers:

**IF/ID:** Holds the fetched instruction word and the value PC+4. The instruction is needed by the decode stage. PC+4 is propagated for use in branch-and-link instructions (JAL/JALR need to save the return address).

**ID/EX:** Holds everything the EX stage needs: register read data for rs1 and rs2, the sign-extended immediate, the destination register index rd, the PC+4 value (for JAL), and all control signals generated by the decode stage (RegWrite, MemRead, MemWrite, ALUSrc, ALUOp, Branch, MemToReg).

**EX/MEM:** Holds the ALU result, the branch target address, the branch condition (zero flag), the store data (rs2 value), the destination register index rd, and the control signals that remain relevant downstream (RegWrite, MemRead, MemWrite, Branch, MemToReg).

**MEM/WB:** Holds the ALU result (pass-through for non-memory instructions), the memory read data (for loads), the destination register index rd, and the control signals RegWrite and MemToReg.

**Note on rd propagation:** The destination register index rd starts in the instruction encoding (bits 11:7). It must be passed through all pipeline registers until WB because the write-back stage must know where to write its result. Forgetting to propagate rd through EX/MEM and MEM/WB is a classic design error.

---

### Question F3
**Why does a single-cycle processor have lower throughput than a pipelined processor, even though each instruction takes more total cycles to complete in the pipelined version?**

**Answer:**

This is the distinction between **latency** and **throughput**.

- **Latency:** How many cycles does one instruction take from start to finish?
  - Single-cycle: 1 cycle (but that cycle is very long, e.g., 880 ps)
  - 5-stage pipeline: 5 cycles (each cycle is short, e.g., 200 ps)

- **Throughput:** How many instructions complete per unit time?
  - Single-cycle: 1 instruction / 880 ps = 1.14 billion instructions per second
  - Pipelined (ideal): 1 instruction / 200 ps = 5 billion instructions per second

The pipeline increases throughput by working on five instructions simultaneously. Once the pipeline is full, one instruction completes every clock cycle. The per-instruction latency is longer (5 cycles at 200 ps = 1000 ps vs. 1 cycle at 880 ps), but five instructions are overlapped, so the effective throughput is much higher.

**Analogy:** A car assembly line has higher throughput than a single workstation where one worker does everything, even though each car takes longer to travel the full assembly line than if one expert assembled it from start to finish alone.

---

### Question F4
**What does the control unit in the ID stage produce, and which signals are used by which downstream stages?**

**Answer:**

The ID stage control unit decodes the opcode (and sometimes funct3/funct7) to produce a set of control signals. These are stored in the ID/EX pipeline register and then selectively passed forward:

```
Signal      Used in   Purpose
----------  --------  -----------------------------------------------------
RegWrite    WB        Whether to write the result to the register file
MemToReg    WB        Select ALU result or memory read data as write-back value
MemRead     MEM       Enable data memory read (assert for LW)
MemWrite    MEM       Enable data memory write (assert for SW)
Branch      MEM/EX    Enable branch condition check and PC update
ALUSrc      EX        Select rs2 or sign-extended immediate as ALU input B
ALUOp[1:0]  EX        Guide the ALU control unit (distinguishes R-type, load/store, branch)
```

Signals consumed in EX (ALUSrc, ALUOp) are not passed to EX/MEM. Signals consumed in MEM (MemRead, MemWrite, Branch) are not passed to MEM/WB. Only RegWrite and MemToReg survive all the way to WB.

**Interview insight:** Candidates are often asked to draw the control signal flow on a pipeline diagram. The key insight is that control signals travel with their instruction through the pipeline, not separately.

---

## Tier 2 — Intermediate

### Question I1
**A RISC-V processor uses a 5-stage pipeline with the branch condition evaluated at the end of the EX stage. The instruction after a branch is always fetched. Describe the exact sequence of events when a branch is taken, and calculate the branch penalty assuming a predict-not-taken scheme.**

**Answer:**

Pipeline diagram for a taken branch (BEQ x1, x2, target — assume x1 == x2):

```
Cycle:     1    2    3    4    5    6    7    8
BEQ:      IF   ID   EX   MEM  WB
I_next:        IF   ID  [squash]
I_next2:            IF  [squash]
I_target:                IF   ID   EX   MEM  WB

                         ^ Branch resolved in EX, cycle 3
                           PC updated to target for fetch in cycle 4
```

Sequence of events:
1. **Cycle 1 (IF):** BEQ is fetched; PC+4 sent to IF/ID.
2. **Cycle 2 (IF/ID):** BEQ decoded; I_next fetched speculatively (predict not taken).
3. **Cycle 3 (ID/EX → EX):** BEQ executes in EX; ALU computes rs1 - rs2; zero flag = 1 (branch taken); branch target = PC_BEQ + imm computed. I_next_2 fetched speculatively.
4. **End of Cycle 3:** Branch logic detects taken branch. PC is updated to branch target. I_next and I_next2 in IF and ID stages are squashed (their pipeline registers are zeroed, injecting NOP bubbles).
5. **Cycle 4 (IF):** First instruction at branch target is fetched.

**Branch penalty = 2 cycles** (two instructions squashed: the one in IF and the one in ID when the branch resolves in EX).

**CPI impact:**
```
CPI_branch = (fraction of branches) * (taken rate) * (penalty)
           = 0.20 * 0.15 * 2 = 0.06 cycles per instruction added to CPI
```

**If branch resolves in MEM stage instead:** penalty increases to 3 cycles. Resolving branches early (ID or EX) is a key microarchitecture optimisation.

**Common mistake:** Forgetting that with predict-not-taken, the penalty only applies to *taken* branches. Not-taken branches have zero penalty because the speculatively fetched instructions are exactly the right ones.

---

### Question I2
**Describe the datapath modifications required to support the JAL (Jump and Link) instruction in a 5-stage RISC-V pipeline. Which pipeline register must be extended, and why?**

**Answer:**

JAL performs two operations:
1. Writes PC+4 (the return address) to the destination register rd.
2. Sets PC = PC_JAL + sign_extend(imm20) — an unconditional jump.

**Datapath additions required:**

**In IF/ID:** No change needed; the instruction is fetched normally.

**In ID (decode):** The immediate must be decoded as a J-type immediate (bits scattered across the instruction word, sign-extended to 21 bits). The decoder sets a new control signal `JumpAndLink = 1`.

**PC update:** The branch adder (or a dedicated jump adder) computes PC_JAL + sign_extend(J-imm). The PC multiplexer must be extended to select this target. This is typically done in the ID or EX stage. If done in ID, the penalty is 1 cycle (only the instruction in IF is squashed); if in EX, 2 cycles.

**Return address path:** The value PC+4 must flow through the pipeline to WB as the write-back data for rd. PC+4 is already stored in IF/ID. It must be propagated through ID/EX and EX/MEM alongside rd. In the WB stage, the MemToReg mux must be extended from 2-way to 3-way:

```
Original MemToReg mux (2-way):
  0: ALU result       (R-type, I-type ALU)
  1: Memory read data (LW)

Extended MemToReg mux (3-way) to support JAL:
  00: ALU result
  01: Memory read data
  10: PC+4 (return address for JAL/JALR)
```

**ID/EX pipeline register extension:** Must carry PC+4 (already present if we use it for branch target computation). No new field needed if PC+4 was already propagated; only the MemToReg encoding and write-back mux require change.

**JALR additionally:** Requires that the jump target = rs1 + sign_extend(I-imm), which is an ALU-computed value. The PC update must therefore happen in or after EX, giving a 2-cycle penalty minimum.

---

### Question I3
**A designer proposes adding a seventh pipeline stage, splitting the MEM stage into "Address Calc" and "Memory Access". What are the trade-offs? Under what circumstances is this beneficial?**

**Answer:**

**Current 5-stage MEM stage work:**
```
MEM stage (combined):
  1. Use ALU result as memory address            ~50 ps (already done in EX)
  2. Data cache tag lookup + way select          ~120 ps
  3. Data array read + hit/miss determination   ~150 ps
  4. Mux output (hit data vs. stall for miss)    ~30 ps
  Total: ~350 ps -- often the bottleneck
```

**After splitting into 6 stages (inserting MA between EX and MEM):**
```
Stage MA (Memory Address / Tag Lookup):
  Present address to cache; initiate tag comparison   ~175 ps

Stage MEM (Memory Data):
  Read data array after hit confirmed; output data    ~175 ps
  Total across both stages: same 350 ps, but each stage is ~175 ps
```

**Benefits:**
1. **Higher clock frequency:** If MEM was the bottleneck at 350 ps and other stages are ~200 ps, splitting MEM allows a ~175 ps clock period — a significant frequency increase.
2. **Better load-to-use forwarding:** With the extra stage, forwarding paths are slightly longer, but cache hit data is available earlier relative to the EX stage of the consuming instruction. In practice, the load-use penalty may increase to 2 stall cycles with a 6-stage pipeline.

**Costs:**
1. **Increased load-use penalty:** A load instruction now takes 2 additional cycles before its result is available in WB. With 5 stages, the load-use penalty is 1 cycle; with the extra MEM split, it may become 2 cycles. This increases CPI for load-heavy code.
2. **Increased branch penalty:** If branch resolution is in the original EX stage, the penalty remains 2 cycles. But if branch outcomes depend on loads (load then branch), the penalty grows.
3. **Deeper pipeline misprediction penalty:** Any control-flow disruption flushes more stages, increasing wasted work.
4. **Pipeline register overhead:** One extra set of flip-flops across the full datapath width adds area and a small power cost.

**When it is beneficial:**
- Clock frequency is limited by the data cache access time, and the design targets a high-frequency server or mobile application processor.
- The workload is not load-intensive (scientific HPC, SIMD-heavy code where loads are amortised).
- A high-accuracy branch predictor can absorb the extra misprediction cycles.
- ARM Cortex-A53 (8-stage in-order) and similar designs use this approach.

---

### Question I4
**Explain the concept of structural hazards. Give two examples specific to a 5-stage RISC-V pipeline and explain how each is resolved.**

**Answer:**

A **structural hazard** occurs when two instructions in different pipeline stages simultaneously require the same hardware resource, and the hardware has only one instance of that resource.

**Example 1 — Single-port memory (combined instruction and data memory):**

If the processor uses a single unified memory for both instructions and data, then in cycle N a load instruction in the MEM stage reads data memory while the instruction fetch unit needs to read instruction memory for a new instruction in the IF stage. Both need the memory simultaneously.

```
Cycle N:
  Load in MEM: reads data from address 0x2000_0000
  New instr in IF: needs to read instruction at PC = 0x0000_1008
  -- conflict on single memory port
```

**Resolution:** Use separate instruction cache (I$) and data cache (D$) — the Harvard architecture model. RISC-V designs universally adopt this; the unified memory conflict is a textbook example that does not occur in real implementations that have separate I$ and D$. Alternatively: add a read port (expensive) or stall IF when MEM accesses memory (CPI cost).

**Example 2 — Single-port register file with simultaneous WB write and ID read:**

The ID stage reads two source registers at the same time as WB writes a destination register. If the register file has only one read/write port:

```
Cycle 5:
  ADD x3, x1, x2 in WB: writes result to x3
  SUB x5, x3, x4 in ID: needs to read x3
  -- if single port, cannot read and write simultaneously
```

**Resolution:** Most register file implementations have two read ports and one write port (or more commonly, two read ports and one write port in a 3-port design). The write occurs early in the clock cycle (first half) and the read occurs late (second half), using the same port sequentially within one cycle. Alternatively, the forwarding network handles the ID/WB case without a register file conflict by forwarding the WB result directly to the ID/EX pipeline register.

**Key distinction:** Structural hazards are about **hardware resource conflicts**, not data dependencies. The solution is always either more hardware (additional ports, separate memories) or scheduling (stalls).

---

## Tier 3 — Advanced

### Question A1
**Describe a complete CPI analysis for the following RISC-V 5-stage pipeline workload. Show all calculations:**

**Pipeline configuration:**
- Branch resolved at end of EX stage, predict-not-taken
- Load-use: 1-cycle stall inserted by hardware (no forwarding past load to immediately following instruction)
- Instruction cache: 96% hit rate, 20-cycle miss penalty
- Data cache: 94% hit rate for loads/stores, 20-cycle miss penalty

**Workload profile:**
- 22% branches, 60% taken
- 25% loads, 8% stores
- 5% of all instructions are immediately followed by a dependent instruction (load-use pairs)

**Answer:**

**CPI = 1 + stall_branch + stall_load_use + stall_I$_miss + stall_D$_miss**

**Branch stalls (predict-not-taken, 2-cycle penalty on taken branches):**
```
Stall cycles per instruction from branches:
  = fraction_branches * branch_taken_rate * penalty
  = 0.22 * 0.60 * 2
  = 0.264 cycles/instruction
```

**Load-use stalls (1-cycle penalty per load-use pair):**
```
  = fraction_load_use_pairs * 1 cycle
  = 0.05 * 1
  = 0.05 cycles/instruction
```

**Instruction cache miss stalls:**
```
  Miss rate = 1 - 0.96 = 0.04
  Every instruction causes an IF stage operation.
  Stall cycles per instruction from I$ misses:
  = miss_rate * miss_penalty
  = 0.04 * 20
  = 0.80 cycles/instruction
```

**Data cache miss stalls:**
```
  Memory instruction fraction = 0.25 (loads) + 0.08 (stores) = 0.33
  Miss rate = 1 - 0.94 = 0.06
  Stall cycles per instruction from D$ misses:
  = fraction_memory * miss_rate * miss_penalty
  = 0.33 * 0.06 * 20
  = 0.396 cycles/instruction
```

**Total CPI:**
```
CPI = 1.000 (ideal)
    + 0.264 (branches)
    + 0.050 (load-use)
    + 0.800 (I$ misses)
    + 0.396 (D$ misses)
    = 2.510 cycles/instruction
```

**Interpretation:** The cache misses dominate, contributing 1.196 out of 1.510 stall cycles (79% of all stalls). Improving cache hit rate from 94% to 98% for data would reduce D$ stalls to 0.132 — saving 0.264 cycles/instruction, a 10.5% CPI reduction. Branch prediction is the second largest contributor; a 2-bit predictor achieving 90% accuracy on taken branches would reduce branch stalls to 0.044.

**Common mistake:** Candidates often forget to account for the fraction of instructions that actually access the data cache (0.33) when computing D$ miss stalls, and instead multiply the miss rate by 1 (every instruction). That gives 0.06 * 20 = 1.2 cycles — a 3x overestimate.

---

### Question A2
**A RISC-V microarchitect is considering moving branch resolution from EX to ID stage. Describe the datapath changes required, the benefit, the hardware cost, and the critical limitation for general branches.**

**Answer:**

**Motivation:** Branch resolution in EX requires 2 stall cycles on a taken branch (the IF and ID stages of the two instructions after the branch are squashed). Moving resolution to ID reduces this to 1 stall cycle.

**Datapath changes required for ID-stage branch resolution:**

1. **Comparator in ID stage:** The branch condition (e.g., rs1 == rs2 for BEQ) requires reading rs1 and rs2 from the register file and comparing them. A dedicated 32-bit comparator must be added to the ID stage. This was previously implicit in the ALU (subtract and check zero flag).

2. **Branch target adder in ID stage:** The branch target = PC + sign_extend(B-imm) must be computed in ID. The PC value is already available in IF/ID. The immediate sign-extension logic must be replicated or moved to ID.

3. **PC multiplexer update:** The mux selecting the next PC must now accept the ID-stage branch target as an input, with the select driven by the branch-taken signal from the ID comparator.

4. **One-cycle squash instead of two:** When a branch is taken, only the instruction currently in IF is squashed (one bubble injected). The instruction in ID is the branch itself and has already been used.

**Benefit:**
```
Old penalty (branch in EX): 2 cycles on taken branch
New penalty (branch in ID): 1 cycle on taken branch

CPI improvement = fraction_branches * taken_rate * (2 - 1)
                = 0.22 * 0.60 * 1 = 0.132 cycles/instruction
```

**Hardware cost:**
- Additional 32-bit comparator in ID stage.
- Branch target adder (may share the existing adder if PC+4 computation is refactored).
- Slightly increased ID stage critical path delay.

**Critical limitation — data hazards on branch operands:**

If a branch instruction depends on the result of an immediately preceding instruction, that result is not yet available in the register file when the branch is in ID:

```
ADD x1, x2, x3    -- in EX when branch is in ID; result not written to RF yet
BEQ x1, x4, label -- in ID; needs x1, but ADD result is still in EX pipeline reg
```

**Resolution options:**
1. **Stall:** Detect that the branch source register matches a pending WB target; stall the branch in ID until the data is available. This effectively eliminates the benefit for many branches in practice.
2. **Forwarding into ID:** Forward from EX/MEM or MEM/WB pipeline registers into the ID comparator. This requires forwarding paths to the ID stage, increasing wiring complexity.

**Conclusion:** ID-stage branch resolution is beneficial when the branch operands are not data-dependent on immediately preceding instructions. Most in-order cores (e.g., RISC-V PicoRV32, simple MCU cores) resolve branches in ID and accept the occasional extra stall for data-dependent branches. High-performance cores use full branch prediction to avoid branch penalties entirely.

---

### Question A3
**Explain pipeline bypassing (forwarding) at the architectural level. Derive the complete set of forwarding conditions for a standard 5-stage pipeline, and explain what makes the load-use case unresolvable by forwarding alone.**

**Answer:**

**Why forwarding is needed:**

The register file is written in WB (cycle N+4 for an instruction that enters EX in cycle N). A dependent instruction that enters EX in cycle N+1 needs the result in cycle N+1, but the result will not be in the register file until cycle N+4. Forwarding takes the result directly from the pipeline register where it sits and routes it to the ALU inputs, bypassing the register file.

**Complete forwarding conditions (from EX/MEM and MEM/WB to EX stage inputs):**

```
Let:
  EX/MEM.rd   = destination register of instruction currently in MEM stage
  MEM/WB.rd   = destination register of instruction currently in WB stage
  ID/EX.rs1   = source register 1 of instruction currently entering EX
  ID/EX.rs2   = source register 2 of instruction currently entering EX

EX-to-EX forwarding (from EX/MEM register to ALU input):
  Condition A (forward to ALU input A):
    EX/MEM.RegWrite == 1          AND
    EX/MEM.rd != x0               AND   (x0 is always zero, never a real dependency)
    EX/MEM.rd == ID/EX.rs1
  => ForwardA = 10 (select EX/MEM.ALU_result instead of register file output)

  Condition B (forward to ALU input B):
    EX/MEM.RegWrite == 1          AND
    EX/MEM.rd != x0               AND
    EX/MEM.rd == ID/EX.rs2
  => ForwardB = 10

MEM-to-EX forwarding (from MEM/WB register to ALU input):
  Condition C (forward to ALU input A, lower priority than A):
    MEM/WB.RegWrite == 1          AND
    MEM/WB.rd != x0               AND
    NOT (EX/MEM.RegWrite AND EX/MEM.rd != x0 AND EX/MEM.rd == ID/EX.rs1)  AND
    MEM/WB.rd == ID/EX.rs1
  => ForwardA = 01 (select MEM/WB result)

  Condition D (forward to ALU input B, lower priority than B):
    MEM/WB.RegWrite == 1          AND
    MEM/WB.rd != x0               AND
    NOT (EX/MEM.RegWrite AND EX/MEM.rd != x0 AND EX/MEM.rd == ID/EX.rs2)  AND
    MEM/WB.rd == ID/EX.rs2
  => ForwardB = 01

Default (no forwarding): ForwardA = 00, ForwardB = 00 (use register file output)
```

The priority condition prevents a false stale-value selection when two in-flight instructions both write rd and the consumer depends on the more recent one (EX/MEM has the newer result).

**Why load-use cannot be solved by forwarding alone:**

```
Cycle timeline:
  LW  x1, 0(x2):  IF  ID  EX  MEM  WB
                               ^--- memory read data available HERE (end of MEM, cycle 4)
  ADD x3, x1, x4:     IF  ID  EX   MEM  WB
                           ^--- needs x1 HERE (start of EX, cycle 3)
```

The LW result comes out of the data memory at the *end* of the MEM stage (cycle 4 relative to LW's IF). The dependent ADD needs it at the *start* of its EX stage (cycle 3). The data is needed one full cycle before it is available. No amount of forwarding wiring can deliver data backwards in time.

**The only solution:** Insert a one-cycle stall (bubble) between LW and ADD. This is detected in the ID stage by checking:

```
Load-use stall condition:
  ID/EX.MemRead == 1    AND
  (ID/EX.rd == IF/ID.rs1  OR  ID/EX.rd == IF/ID.rs2)

Action: stall IF and ID (hold pipeline registers), inject NOP bubble into EX.
```

After the stall, the LW result is now in EX/MEM and can be forwarded normally via Condition A or B above.
