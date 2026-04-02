# Quiz: Microarchitecture

## Instructions

15 multiple-choice questions covering RISC-V pipeline design, data and control hazards,
forwarding logic, branch prediction, and out-of-order execution basics. Each question
has exactly one correct answer. Work through all questions before checking the answer
key at the end.

Difficulty distribution: Questions 1-5 Fundamentals, Questions 6-11 Intermediate,
Questions 12-15 Advanced.

---

## Questions

### Q1 (Fundamentals)

A classic in-order RISC-V pipeline has five stages. What is the correct order of those
stages from first to last?

- A) Fetch, Decode, Execute, Write Back, Memory Access
- B) Fetch, Decode, Execute, Memory Access, Write Back
- C) Decode, Fetch, Execute, Memory Access, Write Back
- D) Fetch, Execute, Decode, Memory Access, Write Back

---

### Q2 (Fundamentals)

What is the ideal CPI (Cycles Per Instruction) of a perfectly pipelined in-order
processor once the pipeline is fully filled, assuming no hazards or cache misses?

- A) Equal to the number of pipeline stages
- B) 0.5
- C) 1
- D) It depends on the instruction mix.

---

### Q3 (Fundamentals)

Which type of pipeline hazard occurs when an instruction requires a result that has
not yet been written back by a preceding instruction in the pipeline?

- A) Structural hazard
- B) Control hazard
- C) Data hazard (specifically a RAW — Read After Write — dependency)
- D) Resource hazard

---

### Q4 (Fundamentals)

In a 5-stage RISC-V pipeline, the `LW x1, 0(x2)` instruction is immediately followed
by `ADD x3, x1, x4`. Why does this create a special hazard that cannot be resolved
by forwarding alone?

- A) `LW` uses a different execution unit than `ADD`, causing a structural hazard.
- B) The load data is not available until the end of the MEM stage, which is one cycle
     after the `ADD` needs it at the start of its EX stage — data arrives too late
     for any forwarding path to deliver it in time.
- C) The register file has insufficient read ports to handle both instructions
     simultaneously.
- D) `LW` and `ADD` cannot be paired because they have different opcodes.

---

### Q5 (Fundamentals)

What does a 2-bit saturating counter branch predictor do that a 1-bit predictor does not?

- A) It predicts branch direction using the target address rather than the branch PC.
- B) It requires two consecutive mispredictions before switching its predicted direction,
     reducing thrashing when a loop body runs with a taken branch but exits once.
- C) It uses two bits to encode four states, allowing it to predict indirect jumps
     in addition to conditional branches.
- D) It shares a single counter between all branches, giving a global prediction.

---

### Q6 (Intermediate)

A 5-stage pipeline with forwarding encounters the following sequence:

```
I1: ADD  x5, x1, x2    # writes x5
I2: SUB  x6, x5, x3    # reads x5 (depends on I1)
I3: AND  x7, x5, x4    # reads x5 (depends on I1)
```

How many stall cycles are inserted by the hazard detection unit, and from which
pipeline register is the forwarded value sourced for each dependent instruction?

- A) 2 stall cycles total; I2 forwards from MEM/WB, I3 forwards from WB.
- B) 0 stall cycles; I2 forwards from EX/MEM, I3 forwards from MEM/WB.
- C) 1 stall cycle for I2 only; I3 forwards from EX/MEM without stalling.
- D) 0 stall cycles; both I2 and I3 forward from EX/MEM.

---

### Q7 (Intermediate)

Consider a branch instruction resolved at the end of the EX stage in a 5-stage
pipeline, using a predict-not-taken scheme. The branch is taken. How many instructions
are squashed, and at what point in the pipeline are they squashed?

- A) One instruction (the one in IF) is squashed when the branch reaches MEM.
- B) Two instructions (those in IF and ID) are squashed when the branch resolves in EX
     at the end of cycle 3 relative to the branch fetch.
- C) Three instructions are squashed; the pipeline is flushed back to the branch itself.
- D) No instructions are squashed; predict-not-taken means the processor waits before
     fetching the next instruction.

---

### Q8 (Intermediate)

Register renaming is a key technique in out-of-order processors. What class of
dependency does it eliminate, and how?

- A) It eliminates RAW (Read After Write) dependencies by duplicating the register file
     and having each instruction write to its own private copy.
- B) It eliminates WAR (Write After Read) and WAW (Write After Write) false dependencies
     by mapping architectural registers to a larger pool of physical registers, ensuring
     that no two in-flight instructions share a physical destination register unless they
     truly depend on each other.
- C) It eliminates structural hazards by providing more execution units, each with its
     own register file.
- D) It eliminates control hazards by predicting which register a branch reads and
     pre-fetching its value.

---

### Q9 (Intermediate)

In an out-of-order processor, what is the purpose of the Reorder Buffer (ROB)?

- A) To buffer instructions waiting for their source operands to become available,
     allowing out-of-order dispatch to execution units.
- B) To maintain program order for in-order commit, ensuring that exceptions are
     reported precisely and that architectural state is updated in the order the
     programmer specified.
- C) To cache the results of recently executed instructions, reducing the number of
     times the same computation is repeated.
- D) To hold the branch prediction state for each in-flight instruction, allowing
     fast recovery on misprediction.

---

### Q10 (Intermediate)

A Return Address Stack (RAS) is a microarchitectural structure used in branch
prediction. What specific instruction class does it target and why is a standard
branch predictor insufficient for this class?

- A) It targets indirect jumps (JALR with non-zero immediate) because their targets
     vary with runtime data and cannot be predicted from history tables.
- B) It targets function return instructions (JALR with rd=x0 and rs1=ra, i.e., `RET`)
     because the return target is different every time a function is called from a
     different call site, making history-based predictors inaccurate.
- C) It targets conditional branches with taken rates near 50%, where standard 2-bit
     predictors perform poorly.
- D) It targets AUIPC+JALR pairs used for position-independent calls, which require
     knowing the current PC at prediction time.

---

### Q11 (Intermediate)

An out-of-order RISC-V core uses Tomasulo's algorithm with a unified reservation
station. Instructions are dispatched out of order to execution units but must commit
in order. Which of the following sequences can execute out of order on this core?

```
I1: LW   x1, 0(x10)    # load; assume cache miss, takes 20 cycles
I2: ADD  x2, x3, x4    # no dependency on x1
I3: ADD  x5, x1, x2    # depends on x1 (from I1) and x2 (from I2)
```

- A) I3 can execute before I1 completes, as long as x2 is available.
- B) I2 can execute before I1 completes; I3 must wait for both I1 and I2 to complete.
- C) No out-of-order execution is possible because I3 depends on both I1 and I2.
- D) I1 and I2 can execute in parallel, but I3 must wait in program order behind I1.

---

### Q12 (Advanced)

A designer wants to reduce the load-use penalty in a 5-stage RISC-V pipeline from
1 cycle to 0 cycles. Describe what would be required and why this is generally not
done.

- A) Moving the register file write to the EX stage eliminates the load-use stall
     because the result is available one stage earlier. This is standard practice in
     modern cores.
- B) Eliminating the load-use stall entirely would require the memory read data to be
     available at the start of EX for the dependent instruction — meaning the cache
     response would need to arrive before the dependent instruction's EX stage begins.
     This is a temporal impossibility for a single-cycle cache access: the MEM stage
     must complete before EX of the next instruction begins in a normal pipeline.
     It would require either a zero-latency cache (impractical) or speculative
     execution with value prediction, which adds complexity and risk.
- C) Adding a second data cache port eliminates the structural hazard that causes
     load-use stalls, reducing the penalty to 0 cycles with no other changes.
- D) Reordering the MEM and WB stages so that WB comes before MEM removes the
     load-use dependency and eliminates the stall cycle.

---

### Q13 (Advanced)

A 5-stage in-order pipeline processes the following instruction sequence. With full
forwarding enabled and a 1-cycle load-use stall, draw the pipeline execution diagram
and calculate the total number of cycles to complete all four instructions (from the
cycle I1 enters IF until the cycle I4 exits WB).

```
I1: LW   x1, 0(x2)     # load
I2: ADD  x3, x1, x4    # RAW on x1 from I1 (load-use hazard)
I3: SUB  x5, x3, x6    # RAW on x3 from I2
I4: OR   x7, x5, x8    # RAW on x5 from I3
```

- A) 8 cycles
- B) 9 cycles
- C) 10 cycles
- D) 12 cycles

---

### Q14 (Advanced)

An out-of-order RISC-V processor encounters a precise exception on instruction I5
while instructions I6, I7, and I8 have already been issued out of order and some have
produced results in the ROB. What must the processor do, and why is this harder than
exception handling in an in-order pipeline?

- A) The processor flushes all instructions after I5 from the pipeline, drains I5 to
     commit, signals the exception, and then begins fetching from the exception handler.
     This is simple because the ROB already maintains program order.
- B) The processor must: (1) allow I1–I4 (those before I5) to commit in order,
     (2) mark I5 as faulting and prevent it from committing, (3) squash I6–I8 and all
     later in-flight instructions, (4) restore the precise architectural register state
     to what it was just before I5 executed, (5) redirect fetch to the exception vector.
     This is harder than in-order because the results of I6–I8 may have already been
     computed and written to physical registers — the ROB must track the precise state
     boundary and the register rename map must be rolled back.
- C) The processor re-executes all instructions from the point of the exception in
     program order, using the in-order retirement log to replay instructions up to
     I5 and then invoking the handler.
- D) Precise exceptions are not supported in out-of-order processors; the RISC-V
     specification allows imprecise exception reporting for performance.

---

### Q15 (Advanced)

A RISC-V out-of-order processor uses a TAGE (TAgged GEometric history length)
branch predictor. What is the fundamental advantage of TAGE over a simple global
history predictor, and what does the "geometric" part of the name refer to?

- A) TAGE uses tagged entries to avoid aliasing, and the geometric naming refers to the
     geometric growth in the number of predictor tables as branch history increases,
     allowing fine-grained partitioning of the branch history register.
- B) TAGE uses multiple predictor tables indexed by geometrically increasing lengths of
     global branch history (e.g., 2, 4, 8, 16, 32 branches of history). Longer-history
     tables capture complex patterns that shorter histories cannot, while shorter tables
     provide coverage when the longer-history entry is not yet trained. Each table
     entry includes a tag to prevent aliasing between branches that happen to map to the
     same index. The predictor uses the longest-history table that has a matching tagged
     entry, falling back to shorter histories or a base predictor.
- C) TAGE replaces the branch history register entirely with a geometric model of loop
     iteration counts, making it particularly effective for loop-dominated code.
- D) TAGE uses a tree of binary predictors arranged in a geometric binary tree, where
     each level of the tree handles one bit of the branch outcome history.

---

## Answer Key

| Q  | Answer | Difficulty    |
|----|--------|---------------|
| 1  | B      | Fundamentals  |
| 2  | C      | Fundamentals  |
| 3  | C      | Fundamentals  |
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

### Q1 - Answer: B (IF, ID, EX, MEM, WB)

The canonical five-stage RISC-V pipeline order is:
1. **IF** (Instruction Fetch): Read the instruction word from the instruction cache.
2. **ID** (Instruction Decode / Register Read): Decode the instruction and read source registers.
3. **EX** (Execute): ALU operation, branch condition evaluation, address computation.
4. **MEM** (Memory Access): Read from or write to the data cache.
5. **WB** (Write Back): Write the result to the destination register.

- **A** has WB before MEM, which would mean the processor writes a load result before
  it has been fetched from memory — logically impossible.
- **C** has Decode before Fetch, which is impossible since the instruction is not known
  until it is fetched.
- **D** has Execute before Decode, which is impossible since the operation cannot be
  performed before the instruction is decoded.

---

### Q2 - Answer: C (1 CPI)

Once the pipeline is fully filled (after the first N-1 cycles for an N-stage pipeline),
one instruction completes (exits the WB stage) every clock cycle. This gives CPI = 1
in the ideal case. This is the fundamental throughput advantage of pipelining: even
though each individual instruction takes N cycles to complete (high latency), the
throughput is 1 instruction per cycle.

- **A** is wrong: N-cycle CPI would mean instructions are not overlapped at all, which
  is the single-cycle case.
- **B (0.5)** would require superscalar execution (completing more than one instruction
  per cycle) — a different architecture class.
- **D** is wrong: in the ideal (no-hazard) case, CPI is exactly 1 regardless of the
  instruction mix.

---

### Q3 - Answer: C (RAW data hazard)

A Read After Write (RAW) hazard occurs when instruction B reads a register that
instruction A has not yet written back. This is a true data dependency: B genuinely
needs A's result. The other hazard types are:

- **Structural hazard**: two instructions compete for the same hardware resource
  simultaneously (e.g., both needing the single memory port).
- **Control hazard**: the next instruction address is not yet known (branch not resolved).
- **WAR and WAW**: false dependencies that arise in out-of-order execution; they do not
  occur in in-order pipelines.

"Resource hazard" in option D is not a standard classification term.

---

### Q4 - Answer: B (load-use timing impossibility)

In a 5-stage pipeline, `LW` reads the data memory in the MEM stage and the result is
available at the end of MEM. The immediately following `ADD` enters the EX stage one
cycle before `LW` completes MEM. Forwarding can deliver results from EX/MEM or MEM/WB
pipeline registers to the EX stage inputs, but those registers are written at the end
of each cycle — the `LW` result lands in EX/MEM at the end of `LW`'s MEM stage, which
is the same cycle that `ADD` needs the value at the start of EX. One stall cycle
bridges this gap, after which the result can be forwarded from the EX/MEM register.

- **A** is wrong: `LW` and `ADD` use different execution units but that is not the
  source of this hazard.
- **C** is wrong: the register file port count is a structural hazard, unrelated to
  load-use.
- **D** is wrong: differing opcodes never prevent pipelining of sequential instructions.

---

### Q5 - Answer: B (two mispredictions needed to switch prediction)

A 1-bit predictor switches its prediction immediately on a single misprediction. For a
loop that executes 100 times and then exits, the predictor mispredicts twice per loop
invocation: once on the exit iteration (predicts taken, but branch is not taken) and
once on the first iteration of the next loop entry (predicts not-taken, but the branch
is taken again). A 2-bit predictor has four states (strongly taken, weakly taken,
weakly not-taken, strongly not-taken). A single misprediction moves it only one state
towards the opposite direction; it takes two consecutive mispredictions to cross the
midpoint and change the predicted direction. For the loop example, only one
misprediction occurs per loop invocation (the exit iteration), which is not enough to
flip the prediction — the predictor remains in "taken" territory.

- **A** is wrong: neither 1-bit nor 2-bit predictors use the branch target for direction
  prediction; that is the role of a Branch Target Buffer (BTB).
- **C** is wrong: 2-bit predictors are not designed for indirect jump prediction.
- **D** is wrong: a global predictor (e.g., gshare) shares state across branches, but
  a 2-bit counter is a per-branch structure; sharing is a separate design choice.

---

### Q6 - Answer: B (0 stall cycles, EX/MEM then MEM/WB forwarding)

With full forwarding: when I2 is in EX, I1 is in MEM. The ALU result of I1 is in the
EX/MEM pipeline register. The forwarding unit detects `EX/MEM.rd == ID/EX.rs1` (both
are x5) and forwards from EX/MEM to the I2 ALU input — 0 stall cycles.

When I3 is in EX, I1 is in WB. I1's result is in MEM/WB. The forwarding unit detects
`MEM/WB.rd == ID/EX.rs1` (x5) and forwards from MEM/WB — again 0 stall cycles.

The forwarding network can cover both a 1-cycle separation (EX/MEM forward) and a
2-cycle separation (MEM/WB forward) without any stalls.

- **A** is wrong: full forwarding eliminates these stalls entirely.
- **C and D** are wrong: no stalls are needed for either instruction with full forwarding.

---

### Q7 - Answer: B (2 instructions squashed in EX)

With predict-not-taken, the processor fetches sequentially after the branch. The branch
is in EX in cycle 3 (IF in cycle 1, ID in cycle 2, EX in cycle 3). At the end of
cycle 3, the branch is resolved as taken. At this point:
- The instruction at PC+4 is in ID (fetched in cycle 2).
- The instruction at PC+8 is in IF (fetched in cycle 3).

Both are on the wrong path. They are squashed by zeroing (flushing) the IF/ID and
ID/EX pipeline registers, injecting two NOP bubbles. The correct target instruction
is fetched in cycle 4.

- **A** is wrong: with branch resolution in EX (not MEM), squashing happens one cycle
  earlier; also, one instruction in MEM is the branch itself, not a squashed instruction.
- **C** is wrong: flushing back to the branch itself would discard the branch, which
  has already committed its decode/execute work.
- **D** is wrong: predict-not-taken fetches speculatively; it does not wait. The penalty
  occurs only when the prediction is wrong (i.e., the branch is taken).

---

### Q8 - Answer: B (eliminates WAR and WAW false dependencies)

Register renaming addresses false dependencies — hazards that arise not from genuine
data flow but from reuse of the same architectural register name. There are two kinds:

- **WAR (Write After Read)**: instruction B writes a register that instruction A reads.
  In a reordered execution, if B completes before A reads, A gets the wrong value.
- **WAW (Write After Write)**: instructions A and B both write the same register. If B
  commits before A, A's later write is lost.

By mapping each architectural register write to a fresh physical register, renaming
ensures that no two instructions in flight write the same physical register, eliminating
both WAR and WAW hazards without ordering constraints.

RAW dependencies (true dependencies) cannot be eliminated by renaming — instruction B
genuinely needs A's output, and renaming does not change that data flow requirement.

- **A** is wrong: renaming does not resolve RAW hazards.
- **C** is wrong: renaming does not add execution units; it is a register management
  technique.
- **D** is wrong: renaming has no interaction with branch prediction.

---

### Q9 - Answer: B (in-order commit for precise exceptions)

The ROB records instructions in program order as they are fetched and decoded. Instructions
execute out of order and write results back to the ROB entry (not to the architectural
register file directly). Commit (also called retirement) happens in order from the head
of the ROB: an instruction commits only when it is at the head and its execution is
complete. This guarantees that:

1. The architectural register file always reflects the committed (precise) program state.
2. Exceptions are handled precisely: if instruction I faults, all instructions before I
   have committed, and I and all later instructions can be squashed cleanly.

- **A** describes a reservation station or issue queue, not the ROB.
- **C** describes a result cache or memoisation structure, not the ROB.
- **D** describes branch prediction tables and misprediction recovery buffers, not the ROB.

---

### Q10 - Answer: B (function return instructions)

The `RET` pseudo-instruction (encoded as `JALR x0, x1, 0`) jumps to the value in `ra`
(x1). Every call site that calls a function pushes a different return address onto the
hardware return address stack. When the function executes `RET`, the correct target is
the address pushed by the most recent `JAL`/`JALR` that called into this function. A
standard global history predictor trained on past branch outcomes cannot reliably
predict this because the same `RET` instruction jumps to a different address depending
on the call chain.

The RAS works as a LIFO stack: a `JAL`/`JALR` call pushes the return address, and a
`RET` pops the most recent entry. On a well-tuned stack, return prediction accuracy
exceeds 95%.

- **A** is wrong: indirect jumps (JALR in general) need a different structure such as
  an Indirect Branch Target Buffer (IBTB); the RAS specifically targets returns.
- **C** is wrong: 2-bit predictors handle near-50% branches adequately; the RAS problem
  is about target address prediction, not direction prediction.
- **D** is wrong: AUIPC+JALR for PIC calls is a specific form of indirect branch, but
  the classical RAS usage is for function returns, not general PIC calls.

---

### Q11 - Answer: B (I2 executes before I1 completes; I3 waits for both)

In an out-of-order processor, I2 (`ADD x2, x3, x4`) has no dependency on I1's output
(`x1`). It can be dispatched to an integer ALU as soon as its sources (`x3`, `x4`) are
available, even while I1 is stalled waiting for the cache. I2 will complete well before
I1 finishes the 20-cycle cache miss.

I3 (`ADD x5, x1, x2`) depends on both `x1` (from I1) and `x2` (from I2). It must
wait until both are available. Since I2 completes quickly and I1 takes 20 cycles, I3
effectively waits only on I1.

- **A** is wrong: I3 cannot execute before I1 because I3 has a true (RAW) dependency
  on x1.
- **C** is wrong: out-of-order execution is possible; I2 is independent of I1.
- **D** is wrong: I3 can dispatch to a reservation station immediately; it waits there
  for x1 and x2 to be broadcast, not "in program order behind I1."

---

### Q12 - Answer: B (temporal impossibility; requires speculative value prediction)

In a standard pipeline, the load data exits the data cache at the end of the MEM stage.
For the consuming instruction to use this data without stalling, it would need the data
at the start of its own EX stage — but that EX stage occurs in the same cycle as the
LW's MEM stage. The data would need to be read, transmitted, and set up as an ALU input
within zero time, which is not physically achievable with a real memory array.

Eliminating the load-use stall without hardware speculation requires either:
- A truly zero-latency cache (a register file is essentially a zero-latency cache, but
  a 4 KB SRAM cache has non-trivial access time), or
- Value prediction: the processor speculates on the loaded value, computes with it, and
  verifies after the memory read completes — adding verification, squash, and re-execute
  logic.

- **A** is wrong: moving the WB write earlier does not help because the data memory
  result is the bottleneck, not the write-back stage.
- **C** is wrong: the load-use hazard is a timing (data) hazard, not a structural hazard.
  A second cache port would allow a simultaneous load and instruction fetch but would not
  deliver the load result one cycle sooner.
- **D** is wrong: reordering MEM and WB would mean writing back before the load data is
  read, which is logically inconsistent.

---

### Q13 - Answer: B (9 cycles)

Pipeline diagram (stall = bubble inserted, fwd = forwarding used):

```
Cycle:  1    2    3    4    5    6    7    8    9
I1 LW:  IF   ID   EX   MEM  WB
I2 ADD:      IF   ID  [stl] EX   MEM  WB        <- 1 stall cycle (load-use)
I3 SUB:           IF   ID   ID   EX   MEM  WB   <- stalled one cycle with I2
I4 OR:                 IF   IF   ID   EX   MEM  WB <- stalled one cycle with I2
```

Explanation:
- I2 needs x1 from I1's load. The stall unit detects load-use and inserts 1 bubble
  after I2's ID stage. After the stall, x1 is forwarded from EX/MEM (I1 is now in WB,
  result in MEM/WB, which forwards to I2's EX).
- I3 needs x3 from I2. When I3 reaches EX, I2 is in MEM — forwarding from EX/MEM, no
  stall.
- I4 needs x5 from I3. When I4 reaches EX, I3 is in MEM — forwarding from EX/MEM, no
  stall.

I1 enters IF in cycle 1. I4 exits WB in cycle 9. Total = 9 cycles.

- **A (8)** would be correct if there were no stall — but the load-use hazard adds 1
  cycle to the 8-cycle ideal completion time.
- **C (10)** would imply two stall cycles; only one load-use stall occurs here.
- **D (12)** implies multiple stalls with no forwarding — forwarding eliminates all
  stalls except the unavoidable load-use one.

---

### Q14 - Answer: B

In an in-order pipeline, when an exception occurs, all instructions in the pipeline
either belong to the instruction causing the exception or to later instructions, so
flushing all later instructions is straightforward. The precise state is whatever was
committed before the exception instruction.

In an out-of-order processor, the situation is more complex because:
- I6, I7, I8 may have executed and produced results placed in the ROB or physical
  registers.
- The ROB guarantees they have NOT committed (committed = architecturally visible), but
  their physical register file entries may contain results of speculative execution.
- The register rename map must be rolled back to the state it was in just before I5
  issued, so that re-fetching from the exception vector starts with the correct
  architectural register bindings.

The ROB structure enables this: each ROB entry records the old physical register
mapping that was overwritten by that instruction's rename. Rolling back consists of
walking the ROB from the tail (newest) back to I5, restoring the old physical register
mappings in reverse order, and freeing the physical registers allocated speculatively.

- **A** is partially right in describing the outcome, but understates the complexity
  by claiming the ROB "already" handles this simply.
- **C** is wrong: modern OoO processors do not replay from a log; they use the ROB
  rollback mechanism.
- **D** is wrong: RISC-V requires precise exceptions. The spec explicitly mandates that
  the architectural state at the time of an exception is precisely defined.

---

### Q15 - Answer: B

A simple global history predictor uses a single history register of fixed length N and
a pattern history table indexed by the history. It works well for patterns up to length
N, but misses longer patterns and suffers from aliasing (different branches mapping to
the same PHT entry).

TAGE addresses both issues:

1. **Geometric history lengths**: TAGE maintains multiple predictor tables, each indexed
   by a different length of global history. The lengths grow geometrically (e.g., 2, 4,
   8, 16, 32, 64 bits), so the collection spans from short, well-trained patterns to
   long, specialized patterns. The predictor selects the longest history table that has
   a matching tagged entry, providing the most contextually specific prediction available.

2. **Tagged entries**: Each entry in the longer-history tables includes a partial tag
   derived from the branch PC and history. Only entries whose tag matches the current
   branch are used. This prevents aliasing: if a long-history index happens to match
   a different branch's history, the tag mismatch will cause the predictor to fall back
   to a shorter history rather than use a wrong prediction.

- **A** is partially right in describing tags and geometric tables but reverses the
  relationship — the geometric growth is in history lengths (not number of tables as
  history grows), and the description is incomplete.
- **C** is wrong: TAGE does not model loop iteration counts; that is the role of a
  separate loop predictor component sometimes combined with TAGE in a hybrid.
- **D** is wrong: TAGE is not a binary tree of predictors; it is a set of parallel
  tables with fallback logic.
