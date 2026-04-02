# Branch Prediction

## Prerequisites
- The classic 5-stage RISC-V pipeline and control hazards
- Branch instruction encoding in RISC-V (B-type, J-type)
- Basic CPI analysis (see pipeline_design.md)

---

## Concept Reference

### Why Branch Prediction Matters

In a 5-stage pipeline where the branch condition is evaluated in EX (cycle 3 of the branch instruction), a taken branch flushes the two instructions speculatively fetched after the branch. This is a 2-cycle penalty per taken branch.

```
Typical workload: 20% branches, 60% taken.
Without prediction (predict-not-taken):
  CPI_branch = 0.20 * 0.60 * 2 = 0.24 cycles wasted per instruction

In a program executing 1 billion instructions at 1 GHz:
  Wasted cycles = 0.24 * 10^9 = 240 million cycles = 0.24 seconds
```

A modern out-of-order core with a 15-stage pipeline and 15-cycle misprediction penalty makes this worse by an order of magnitude. Branch prediction accuracy is a first-order performance concern.

### Static Branch Prediction

Static prediction makes the same decision for a given branch every time it is encountered, regardless of runtime history. There is no state table to maintain.

```
Common static strategies:
  1. Predict-not-taken:  Always fetch PC+4 (sequential path).
                         Penalty only on taken branches.
                         Simple; no hardware overhead.

  2. Predict-taken:      Always fetch branch target.
                         Target must be known early (requires early target calculation).
                         Good for loops (which are usually taken on the backward edge).

  3. Backward-taken, forward-not-taken (BTFNT):
                         If branch target < PC (backward branch = loop): predict taken.
                         If branch target > PC (forward branch = if/else): predict not taken.
                         Typical accuracy: 65-75%.
                         Requires comparing target address to PC in IF or ID stage.

  4. Compiler-hinted (RISC-V branch hint encoding):
                         Some ISAs encode a prediction hint in unused instruction bits.
                         RISC-V does not currently define branch hint bits in the base ISA,
                         but compressed branch offsets imply direction hints informally.
```

### Dynamic Branch Prediction: 1-Bit Predictor

A 1-bit predictor maintains one bit of state per branch (or per entry in a shared table):

```
State: T (last outcome was Taken) or NT (last outcome was Not Taken)

Transitions:
  State T, actual Taken:     -> stay T  (predict T correctly)
  State T, actual Not Taken: -> go  NT  (misprediction)
  State NT, actual Taken:    -> go  T   (misprediction)
  State NT, actual Not Taken:-> stay NT (predict NT correctly)

Problem with loops:
  A loop that iterates N times (N-1 taken + 1 not-taken at exit):
  1-bit predictor mispredicts TWICE per loop (once entering, once exiting).
  For N=10: misprediction rate = 2/10 = 20% -- surprisingly poor.
```

### Dynamic Branch Prediction: 2-Bit Saturating Counter

A 2-bit predictor adds hysteresis. A single wrong outcome does not immediately flip the prediction:

```
States (2-bit saturating counter):
  11 = Strongly Taken     (predict Taken)
  10 = Weakly Taken       (predict Taken)
  01 = Weakly Not Taken   (predict Not Taken)
  00 = Strongly Not Taken (predict Not Taken)

Transitions on Taken outcome:    increment (saturate at 11)
Transitions on Not Taken outcome: decrement (saturate at 00)

For the loop example (N=10):
  Initial state: 11 (strongly taken)
  Iterations 1-9: Taken -> stays 11, predicted correctly (9 correct)
  Iteration 10 (loop exit, Not Taken): 11->10 (misprediction, but counter is now Weakly Taken)
  Next loop entry (Taken): 10->11 (predicted correctly, Weakly Taken -> Strongly Taken)
  Misprediction rate: 1/10 = 10% vs. 2/10 for 1-bit predictor.
```

The 2-bit predictor is the classic textbook predictor and underlies more complex designs.

### Branch History Table (BHT) / Pattern History Table (PHT)

A real predictor stores one counter per branch, indexed by the branch's PC:

```
BHT organisation:
  Index:   lower bits of the branch PC (e.g., bits [11:2] for a 1K-entry table)
  Entry:   2-bit saturating counter
  Tag:     optionally, upper PC bits to distinguish branches that alias to the same entry

                 PC[11:2]
                    |
                    v
              +----------+
              | 2b ctr 0 |
              | 2b ctr 1 |
              |   ...    |   -> prediction (bit 1 of counter: 1=Taken, 0=NT)
              | 2b ctr N |
              +----------+

Access latency: 1 cycle (critical path: must be available by end of IF for the next cycle's fetch)
```

**Aliasing:** Two branches with the same PC[11:2] but different PC[31:12] share an entry. This introduces interference (wrong-path training). A tagged predictor (TAGE) addresses this.

### Branch Target Buffer (BTB)

Knowing the branch *direction* (taken/not-taken) is not enough — the processor also needs the *target address* to redirect fetch immediately. The BTB provides this:

```
BTB organisation (direct-mapped or set-associative cache of branch targets):
  Index:   lower bits of the branch PC
  Entry:   branch target address (full 32-bit or 64-bit PC)
           + valid bit
           + optionally: branch type (conditional / unconditional / return)

Access:    In the IF stage, the fetched instruction's PC is used to index the BTB
           simultaneously with instruction memory access. If the BTB hits and
           the direction predictor says Taken, the next PC is set to BTB[entry].target.

BTB miss (first time a branch is encountered):
  The BTB does not have a target entry.
  The branch is fetched, decoded, and the target computed normally.
  The BTB is updated with the correct target.
  Penalty on first encounter: same as predict-not-taken (2 cycles for branch resolved in EX).
```

### Return Address Stack (RAS)

Function return instructions (`JALR x0, x1, 0` in RISC-V) are indirect branches — the target is the contents of a register (ra = x1). The BTB cannot predict these well because each call site has a different return address.

The Return Address Stack (RAS) is a small hardware stack (typically 8-16 entries) that mirrors the call stack:

```
CALL detected (JAL/JALR with rd=x1):
  Push current PC+4 (return address) onto the RAS hardware stack.

RETURN detected (JALR with rs1=x1, rd=x0):
  Pop TOS from RAS; use as predicted target address.

Accuracy: near 100% for well-structured code (no setjmp/longjmp, no tail calls).
Overflow: if call depth > RAS size, oldest entries are overwritten (circular buffer).
          Deep recursion causes RAS thrashing.
```

### Global vs. Local History Predictors

**Local history:** Each branch has its own N-bit shift register recording the last N outcomes (T/NT) for that specific branch. The history indexes a pattern history table (PHT) of 2-bit counters.

```
Local history predictor:
  Branch PC -> local history register (e.g., 10-bit: last 10 outcomes)
                 |
                 v
           PHT[history] -> 2-bit counter -> prediction
```

**Global history:** A single N-bit global history register (GHR) records the last N branch outcomes across all branches, XORed with the branch PC to index the PHT.

```
Global history (gshare):
  GHR XOR PC[N-1:0] -> PHT index -> 2-bit counter -> prediction

Advantage: captures correlations between different branches
           (e.g., "if x>0 is true, then the next branch x>1 is likely true too")
```

**Tournament (hybrid) predictor:** Uses both local and global predictors plus a meta-predictor (itself a table of 2-bit counters) that tracks which sub-predictor is more accurate for each branch. The Alpha 21264 used this approach; many modern cores use a variant.

### TAGE Predictor (Tagged Geometric History Length)

TAGE is the dominant predictor in modern high-performance processors:

```
TAGE structure:
  T0: base 2-bit counter table (bimodal), indexed by PC
  T1: tagged table, history length 2,  indexed by PC XOR GHR[1:0]
  T2: tagged table, history length 4,  indexed by PC XOR GHR[3:0]
  T3: tagged table, history length 8,  indexed by PC XOR GHR[7:0]
  T4: tagged table, history length 16, indexed by PC XOR GHR[15:0]
  ...
  Tk: tagged table, history length L(k) = geometric series (~1.4x each level)

Each tagged entry: partial tag + 3-bit counter + useful bit

Prediction: use the longest-history table that has a tag match (most specific predictor).
            Fall back to shorter histories if no tag match.

Accuracy: 95%+ on standard benchmark suites (SPECint).
Hardware cost: ~32-64 KB of SRAM, with complex update logic.
```

### Branch Penalty Calculation

```
For a pipeline where branch is resolved in EX stage (stage 3):
  Penalty on misprediction = 2 cycles (instructions in IF and ID are squashed)

CPI impact:
  CPI_branch = freq_branch * misprediction_rate * penalty_cycles

Example:
  freq_branch        = 0.20   (20% of instructions are branches)
  misprediction_rate = 0.10   (90% accurate predictor)
  penalty            = 2 cycles

  CPI_branch = 0.20 * 0.10 * 2 = 0.04 cycles/instruction

Compare with no predictor (predict-not-taken, 60% taken):
  CPI_branch = 0.20 * 0.60 * 2 = 0.24 cycles/instruction

Improvement: 0.24 - 0.04 = 0.20 cycles/instruction = ~17% overall CPI reduction
  (assuming baseline CPI without branch stalls = 1.05 from other hazards)
```

### Speculative Execution and Misprediction Recovery

When a branch misprediction is detected:
1. All instructions fetched after the branch on the wrong path must be squashed.
2. The PC must be redirected to the correct path.
3. Any architectural state changes from wrong-path instructions must be undone.

In a simple in-order 5-stage pipeline, wrong-path instructions have not yet written back (they are squashed in the IF/ID and ID/EX stages), so recovery is straightforward: zero out those pipeline registers.

In an out-of-order processor, wrong-path instructions may have entered the reorder buffer (ROB). Misprediction recovery involves flushing all ROB entries younger than the branch and restoring the correct register state from a checkpoint.

---

## Tier 1 — Fundamentals

### Question F1
**What is a branch penalty and why does it occur in a pipelined processor? Calculate the branch penalty for a 5-stage RISC-V pipeline where the branch condition is resolved at the end of the EX stage.**

**Answer:**

A branch penalty is the number of useful instruction slots that are wasted (filled with bubbles/NOPs) because the processor fetched instructions from the wrong path after a branch.

**Why it occurs:** The pipeline fetches instructions speculatively before knowing the branch outcome. The branch is in IF at cycle 1. It is not until cycle 3 (end of EX) that the processor knows whether the branch is taken and, if so, what the target address is. In the meantime, cycles 2 and 3 have already fetched the next two sequential instructions.

```
Cycle:    1    2    3    4    5    6    7    8
BEQ:     IF   ID   EX  MEM   WB
I_next:       IF   ID  [squash]               <- fetched speculatively, squashed
I_next2:           IF  [squash]               <- fetched speculatively, squashed
I_target:               IF   ID   EX  MEM  WB <- correct path resumes here
```

When EX detects "branch taken" at the end of cycle 3, two instructions have been fetched in error. Both are squashed (pipeline registers zeroed, injecting NOPs). The instruction at the branch target enters IF in cycle 4.

**Branch penalty = 2 cycles** (2 squashed instructions).

**Important:** With a predict-not-taken scheme, the penalty only applies when the branch is *taken*. If the branch is not taken, the sequentially fetched instructions were exactly the right ones — zero penalty.

---

### Question F2
**Compare 1-bit and 2-bit branch predictors. For a loop that executes 8 iterations, calculate the misprediction rate for each predictor assuming the loop starts in the "Taken" state.**

**Answer:**

**Loop behaviour:** 7 taken iterations, then 1 not-taken (loop exit), then 7 taken again on the next invocation.

**1-bit predictor (starts in state T):**
```
Iteration 1: predict T, actual T -> correct, state stays T
...
Iteration 7: predict T, actual T -> correct, state stays T
Iteration 8: predict T, actual NT -> MISPREDICTION, state -> NT
Next call, iter 1: predict NT, actual T -> MISPREDICTION, state -> T
...
Mispredict 2 per 8 iterations = 2/8 = 25% misprediction rate
```

**2-bit predictor (starts in state 11 = Strongly Taken):**
```
Iterations 1-7: predict T (bit 1 = 1), actual T -> counter stays 11 (saturated)
Iteration 8: predict T, actual NT -> MISPREDICTION, counter 11 -> 10 (Weakly Taken)
Next call, iter 1: predict T (counter 10, bit 1 = 1), actual T -> correct, counter 10 -> 11
...
Mispredict 1 per 8 iterations = 1/8 = 12.5% misprediction rate
```

**Summary:**

| Predictor | Mispredictions per loop | Rate (N=8) |
|-----------|------------------------|------------|
| 1-bit     | 2                      | 25%        |
| 2-bit     | 1                      | 12.5%      |

The 2-bit predictor halves the misprediction rate for loops by requiring two consecutive wrong-direction outcomes before changing the prediction. For larger N, the improvement is even more significant (2/N vs. 1/N).

---

### Question F3
**What is a Branch Target Buffer (BTB) and why is it needed in addition to a direction predictor?**

**Answer:**

A direction predictor tells the processor whether a branch will be taken or not taken. But if the prediction is "taken," the processor must also know *where* to fetch next. Knowing "this branch will be taken" is useless without the target address.

Computing the branch target requires:
1. Fetching the instruction (IF stage).
2. Decoding the immediate field (ID stage).
3. Adding the immediate to the PC.

This means the target is not known until at least the ID stage — too late to redirect fetch to the target in the very next cycle (which is what a 1-cycle fetch redirect requires).

**The BTB solution:** The BTB is a small cache, indexed by the branch instruction's PC, that stores the target address from the *previous execution* of that branch. In the IF stage, the fetched PC is used to look up the BTB simultaneously with instruction memory. If there is a hit, the predicted target address is available at the end of the IF stage — in time to redirect fetch for the next cycle.

```
IF stage (cycle 1):
  - Fetch instruction at current PC
  - Simultaneously: BTB[PC] lookup
    - If hit AND direction predictor says Taken:
        next_PC = BTB[PC].target  (redirect fetch immediately)
    - If miss OR direction predictor says Not Taken:
        next_PC = PC + 4
```

Without a BTB, even a perfect direction predictor still incurs a penalty on taken branches because the target is computed too late. The BTB eliminates this penalty for branches that have been seen before.

**BTB miss on first encounter:** The branch is predicted not-taken (no BTB entry). After the target is computed, the BTB is updated. This is a cold-start penalty.

---

### Question F4
**What is the Return Address Stack (RAS) and what specific RISC-V instruction idiom does it optimise?**

**Answer:**

The RAS is a small hardware stack (typically 4-16 entries) that predicts the target of function return instructions.

**The problem:** In RISC-V, a function call is `JAL ra, offset` (writes return address to ra) and a return is `JALR x0, ra, 0` (jumps to the address in ra). The return is an *indirect* branch — the target is a register value at runtime, not a fixed offset in the instruction encoding. The BTB can store the last-seen target, but a function called from many call sites will have a different return address each time. The BTB would thrash and predict incorrectly most of the time.

**RAS solution:**
- When the processor detects a JAL or JALR instruction with `rd == x1` (the link register convention), it pushes `PC+4` (the return address) onto the hardware RAS stack.
- When it detects JALR with `rs1 == x1` and `rd == x0` (a canonical return), it pops the TOS from the RAS and uses it as the predicted target.

```
Call sequence:
  0x1000: JAL x1, func    -> RAS.push(0x1004)
  0x1004: (instruction after the call)

  ...
  func:
  0x2000: ADDI x10, x10, 1
  0x2004: JALR x0, x1, 0  -> predicted target = RAS.pop() = 0x1004  (correct)
```

**Accuracy:** Near 100% for well-structured code (correctly matched calls and returns). Degrades for non-standard patterns like tail calls, `setjmp`/`longjmp`, or deeply recursive functions that overflow the small RAS.

---

## Tier 2 — Intermediate

### Question I1
**A RISC-V processor runs a workload with the following profile: 18% of instructions are conditional branches, 55% of which are taken. The pipeline resolves branches in EX (2-cycle penalty). Compare the CPI contribution from branches for three predictor schemes: (a) predict-not-taken, (b) 2-bit counter with 85% accuracy, (c) TAGE predictor with 97% accuracy.**

**Answer:**

Let:
- `f_b` = frequency of branches = 0.18
- `p` = misprediction penalty = 2 cycles

**Predict-not-taken (a):**
Mispredictions occur on every taken branch.
```
Misprediction rate = taken rate = 0.55
CPI_branch = f_b * misprediction_rate * penalty
           = 0.18 * 0.55 * 2
           = 0.198 cycles/instruction
```

**2-bit counter, 85% accuracy (b):**
Misprediction rate = 1 - 0.85 = 0.15.
```
CPI_branch = 0.18 * 0.15 * 2
           = 0.054 cycles/instruction
```

**TAGE predictor, 97% accuracy (c):**
Misprediction rate = 1 - 0.97 = 0.03.
```
CPI_branch = 0.18 * 0.03 * 2
           = 0.0108 cycles/instruction
```

**Summary table:**

| Predictor       | Misprediction rate | CPI from branches | Reduction vs. (a) |
|-----------------|--------------------|-------------------|--------------------|
| Predict NT      | 55%                | 0.198             | baseline           |
| 2-bit counter   | 15%                | 0.054             | -0.144 (73%)      |
| TAGE            | 3%                 | 0.011             | -0.187 (94%)      |

**Interpretation:** The TAGE predictor is only marginally better than the 2-bit counter on an already low CPI-branch term. The real benefit shows in deeper pipelines where the misprediction penalty is 15-20 cycles (modern out-of-order cores), making even the difference between 3% and 15% misprediction rate equivalent to 0.18 * 0.12 * 17 = 0.37 cycles/instruction — a major performance factor.

---

### Question I2
**Explain branch misprediction recovery in a 5-stage in-order pipeline. What actions must the hardware take, and in which cycle does each action occur? Extend your answer to identify any cases where recovery is more complex.**

**Answer:**

**Timeline for a taken branch that was predicted not-taken:**

```
Cycle:    1    2    3    4    5    6    7
BEQ:     IF   ID   EX  MEM   WB
I_wrong1:     IF   ID [flush] <- squashed
I_wrong2:          IF [flush] <- squashed
I_correct:              IF   ID   EX  MEM  WB
                   ^
           End of EX (cycle 3): misprediction detected
```

**Cycle-by-cycle recovery actions:**

**End of cycle 3 (EX stage detects taken branch):**
1. The branch condition (zero flag) and taken signal are asserted.
2. The branch target address (from EX/MEM register or computed in EX) is presented to the PC MUX.
3. The `PCSrc` control signal selects the branch target.

**Beginning of cycle 4 (pipeline register updates):**
4. The IF/ID pipeline register is flushed (zeroed). The instruction in IF (which is I_wrong2) is replaced with a NOP bubble.
5. The ID/EX pipeline register is flushed (zeroed). The instruction in ID (which is I_wrong1) is replaced with a NOP bubble.
6. PC is loaded with the branch target address. The correct instruction (I_correct) begins fetching.

**Result:** Cycles 4 and 5 contain bubbles (the two squashed instructions); I_correct enters IF in cycle 4 and proceeds normally.

**Simpler cases (no recovery needed):**
- Branch not taken with predict-not-taken: I_wrong1 and I_wrong2 are the correct sequential instructions. No flush needed, no penalty.

**More complex recovery cases:**

1. **Branch with operand dependency:** If the branch instruction itself depends on a load (load-use hazard before the branch), the pipeline must both stall (for the load-use) and potentially flush (for the branch). The hazard detection and branch resolution logic must handle these simultaneously.

2. **Branch in a delayed-slot architecture:** RISC-V does not use branch delay slots, but some RISC architectures did. In those, the instruction immediately after the branch always executes (it is in the "delay slot"). RISC-V's clean design eliminates this complication.

3. **Exception during a wrong-path instruction:** If a wrong-path instruction (in IF or ID after the branch) causes an exception before the branch resolves, the processor must know that this exception is on the wrong path and suppress it. In a simple 5-stage pipeline this is straightforward because exceptions are not raised until the instruction reaches MEM or WB, by which time the flush has already occurred. In deeper pipelines with earlier exception detection this becomes a correctness concern.

---

### Question I3
**Describe the BTFNT (Backward-Taken, Forward-Not-Taken) static prediction strategy. What information is available at the IF stage to implement it, and what accuracy can it achieve?**

**Answer:**

**Rationale:** The majority of backward branches (where the target address is less than the branch PC) are loop-back edges. Loops are taken most of the time (until the exit condition fires). The majority of forward branches (target > PC) are if-then-else constructs, which are not-taken as often as taken.

**BTFNT rule:**
- If branch target address < branch PC: predict **Taken** (it is probably a loop back-edge).
- If branch target address > branch PC: predict **Not Taken** (it is probably a forward conditional).

**What is available in IF:**
- The branch PC (current PC register value) — available immediately.
- The branch target — requires knowing the immediate offset, which requires at least partial decode of the instruction. The B-type immediate in RISC-V is spread across bits [31], [30:25], [11:8], [7] of the instruction word. The sign bit (bit 31) determines whether the offset is negative (backward branch) or positive (forward branch).

```
RISC-V B-type offset sign determination from instruction word:
  Offset = sign_extend({inst[31], inst[7], inst[30:25], inst[11:8], 1'b0})
  Offset sign = inst[31]
    0 -> positive offset -> forward branch -> predict NOT TAKEN
    1 -> negative offset -> backward branch -> predict TAKEN
```

Critically, only the single bit inst[31] is needed to make the prediction. This can be decoded in the IF stage (or very early in ID) with minimal logic, so the next-PC decision is only delayed by this single-bit decode.

**Typical accuracy:**
- For loop-back edges: ~90% correct (loops iterate many times before exiting).
- For forward branches: ~50-60% correct (highly variable between programs).
- Overall: approximately 65-75% correct on general-purpose code (better than pure predict-not-taken at ~45% for taken-branch-heavy code).

**Comparison:**

| Strategy        | Hardware cost | Typical accuracy |
|-----------------|---------------|------------------|
| Predict NT      | None          | 40-55%           |
| BTFNT           | 1-bit decode  | 65-75%           |
| 2-bit counter   | BHT table     | 82-88%           |
| TAGE            | ~32-64 KB     | 94-97%           |

BTFNT is used in low-power embedded cores (e.g., some RISC-V MCU implementations) where the area for a dynamic predictor table is not justified.

---

## Tier 3 — Advanced

### Question A1
**A RISC-V core uses a gshare predictor: a 12-bit global history register (GHR) XORed with PC[13:2] to index a 4K-entry table of 2-bit saturating counters. Describe (a) the update algorithm, (b) how aliasing degrades accuracy, and (c) how TAGE mitigates aliasing.**

**Answer:**

**Part (a): Gshare update algorithm:**

```
Predict (at IF stage):
  index = GHR[11:0] XOR PC[13:2]
  prediction = PHT[index][1]          -- MSB of 2-bit counter
  predicted_target = BTB[PC].target   -- from BTB

Execute (at EX stage — branch resolves):
  actual_outcome = branch_taken ? 1 : 0

Update:
  if actual_outcome == 1:
      PHT[index] = saturating_increment(PHT[index])  -- max 2'b11
  else:
      PHT[index] = saturating_decrement(PHT[index])  -- min 2'b00

  // Update GHR: shift in actual outcome
  GHR = {GHR[10:0], actual_outcome}

  // If misprediction:
  //   Redirect PC to correct target
  //   Restore GHR to the value it had at the point of this branch,
  //   then shift in the correct outcome.
  //   (GHR speculative update must be undone)
```

**GHR speculative update issue:** The GHR is updated with the *predicted* outcome during fetch (not the actual outcome after EX) to keep the history current for subsequent branch predictions in the same fetch window. When a misprediction is detected, the GHR must be restored to its correct state. This requires saving a "checkpoint" of the GHR at each branch and restoring it on misprediction.

**Part (b): Aliasing in gshare:**

Two different branches (different PCs with different histories) may produce the same index:
```
Branch A: GHR = 0b101010101010, PC_A[13:2] = 0b001010101010
          index_A = 0b100000000000

Branch B: GHR = 0b110000000000, PC_B[13:2] = 0b010000000000
          index_B = 0b100000000000   <- same index as Branch A
```

If A and B have opposite behaviors (A mostly taken, B mostly not-taken), they will interfere destructively. B's not-taken updates will decrement the counter; A's taken updates will increment it. Both predictions become unreliable.

Aliasing is unavoidable in a hash-based structure. With 4K entries and many branches, collisions are frequent. Studies show that aliasing causes 10-30% of gshare's mispredictions.

**Part (c): TAGE mitigation:**

TAGE (Tagged Geometric history length) adds a partial tag to each predictor entry, discriminating between branches that alias to the same index:

```
TAGE entry fields:
  ctr  : 3-bit signed saturating counter (prediction = sign bit)
  tag  : partial tag (e.g., 10 bits from upper PC XOR upper GHR bits)
  u    : 1-bit "useful" flag (entry was recently the provider of a correct prediction)

Lookup:
  Compute index for each table T1...Tk (different history lengths).
  Find the longest table T_j with a matching tag.
  Use T_j's prediction (most specific, least aliasing).

On tag mismatch: fall back to the next-shortest history table.
Final fallback: base bimodal table T0 (no tag, always hits).

Tag miss means the branch has not been "trained" at this history length.
The allocator will attempt to insert the branch on a misprediction.

Useful bit:
  u is set to 1 when this entry is the provider and the prediction was correct.
  Entries with u=0 are candidates for replacement (they are not proving useful).
  Prevents an entry that has been trained for one branch from being evicted by
  an aliasing branch that has not yet proved useful.
```

The tag check converts destructive aliasing into a simple "miss" — the predictor falls back to a lower-confidence, shorter-history prediction rather than using a corrupted counter. This is the fundamental mechanism by which TAGE achieves 95%+ accuracy where gshare achieves 88-92%.

---

### Question A2
**Speculative execution past a branch requires careful handling of exceptions and memory ordering. Describe two classes of correctness hazards introduced by speculative execution, and explain how each is resolved in a RISC-V out-of-order core.**

**Answer:**

**Hazard class 1: Exceptions from wrong-path instructions**

A wrong-path instruction (fetched after a mispredicted branch) may cause an exception — for example, a wrong-path load from an unmapped address would generate a page fault.

If the processor's exception model is not speculation-aware, this could:
- Raise a spurious exception for an instruction that should never have executed.
- Corrupt exception CSRs (mepc, mcause, mtval) with wrong-path values.
- Cause the OS to handle a fault that architecturally does not exist.

**Resolution in a RISC-V OoO core:**

Exceptions are handled through the Reorder Buffer (ROB). Wrong-path instructions are entered into the ROB but are tagged as speculative. When branch misprediction is detected:
1. All ROB entries younger than the branch are flushed (squashed).
2. Any pending exceptions from these flushed instructions are discarded.
3. Exception CSRs (mepc, mcause) are only updated when an instruction retires from the head of the ROB — in program order.

This ensures that no wrong-path exception ever becomes architecturally visible. The RISC-V ISA's precise exception model (exceptions are taken at the faulting instruction; all preceding instructions have completed; no subsequent instructions have modified state) is preserved because architectural state is only committed at ROB retirement.

**Hazard class 2: Memory ordering and memory-side effects on the wrong path**

A wrong-path store instruction should not modify memory — the store was for an instruction that architecturally did not execute. If a wrong-path store writes to a shared memory location before the branch misprediction is discovered:
- Other hardware threads (on a multicore system) could observe the wrong-path write.
- The RISC-V memory consistency model (RVWMO — a weak ordering model) allows stores to be visible to other harts before retirement.

**Resolution:**

Wrong-path stores are held in the store queue and are never issued to the L1 data cache (let alone to the memory bus) until they reach the head of the ROB and commit. The store queue acts as a speculative buffer:

```
OoO core store path:
  Speculative store instruction:
    1. Entry added to store queue with address, data, speculation tag.
    2. Store queue entry is NOT written to cache while the instruction is speculative.
    3. On ROB commit (instruction is now non-speculative): store is released from
       the store queue to the L1 cache write port. Only now is it visible globally.
    4. On branch flush: speculative store queue entries for flushed instructions
       are simply deallocated (the write never happened).
```

**Spectre-class concern:** Even if a wrong-path load (read) does not modify architectural state, it can modify *microarchitectural* state (cache lines are brought in, TLB entries are populated). The RISC-V memory consistency model permits this — cache state is not architecturally visible. However, side-channel attacks (Spectre) exploit exactly this microarchitectural state change. Mitigations include flushing TLB and cache state on context switches and using serialising instructions (`FENCE`, `FENCE.I`) at security boundaries.

---

### Question A3
**Design a CPI model for a RISC-V OoO processor with a 20-stage pipeline, a tournament branch predictor with 93% accuracy, and ROB capacity of 192 entries. The workload has: 22% branches, 28% loads, 10% stores. L1 I-cache 98% hit, L1 D-cache 95% hit, L2 50-cycle miss penalty. Show how branch mispredictions can serialise the ROB and calculate the effective throughput degradation.**

**Answer:**

**Pipeline and predictor parameters:**
```
Frontend fetch width:      4 instructions/cycle
Branch misprediction penalty: 20 cycles (pipeline depth)
Misprediction rate:           7% (1 - 0.93)
ROB size:                     192 entries
L1 I-cache miss penalty:      8 cycles (to L2 hit)
L1 D-cache miss penalty:      8 cycles (to L2 hit)
L2 miss penalty:              50 cycles (to DRAM)
L1 I-cache miss rate:         2%
L1 D-cache miss rate:         5%
L2 hit rate (given L1 miss):  70%
```

**Branch misprediction CPI contribution:**
```
CPI_branch = freq_branch * misprediction_rate * penalty
           = 0.22 * 0.07 * 20
           = 0.308 cycles/instruction
```

**ROB serialisation effect:** When a branch mispredicts:
1. All 20 pipeline stages are flushed.
2. The ROB is partially drained: instructions older than the branch commit; those younger are squashed.
3. During the 20-cycle recovery, the ROB is being drained; no new instructions commit.
4. The processor effectively stalls for the duration of the flush + refill.

Average instructions in-flight at time of misprediction (ROB fill = ~70% typical):
```
In-flight at misprediction: ~0.70 * 192 = 134 instructions (average occupancy)
Younger than the branch (must be squashed): ~67 instructions (half of in-flight)
```

The squash itself takes ~5 cycles (clearing 67 ROB entries at ~15 per cycle). Combined with the 20-cycle fetch redirect latency, total recovery ≈ 20 cycles. This matches the raw misprediction penalty figure used above.

**I-cache miss CPI:**
```
L1 I-cache miss -> L2 hit:   0.02 * 0.70 * 8  = 0.112
L1 I-cache miss -> L2 miss:  0.02 * 0.30 * 50 = 0.300
CPI_Icache = 0.112 + 0.300 = 0.412 cycles/instruction
```

**D-cache miss CPI (only loads and stores access D-cache):**
```
Memory instruction freq: 0.28 + 0.10 = 0.38
L1 D-cache miss -> L2 hit:   0.38 * 0.05 * 0.70 * 8  = 0.1064
L1 D-cache miss -> L2 miss:  0.38 * 0.05 * 0.30 * 50 = 0.2850
CPI_Dcache = 0.1064 + 0.2850 = 0.391 cycles/instruction
```

**Total CPI (with IPC upper bound from fetch width):**
```
IPC limit from 4-wide fetch: IPC_max = 4.0 (CPI_min = 0.25)

CPI = CPI_ideal + CPI_branch + CPI_Icache + CPI_Dcache
    = 0.25      + 0.308      + 0.412      + 0.391
    = 1.361 cycles/instruction
IPC = 1 / 1.361 = 0.735 instructions/cycle
```

**Throughput degradation from branches alone:**
```
Without branch mispredictions: CPI = 0.25 + 0 + 0.412 + 0.391 = 1.053
With branch mispredictions:    CPI = 1.361
Degradation = (1.361 - 1.053) / 1.053 = 29.2% throughput reduction from branches
```

**Key insight:** Memory latency (I-cache + D-cache) contributes more (0.803) than branch mispredictions (0.308) to the total stall CPI. Improving branch prediction accuracy from 93% to 99% would only save 0.22 * 0.06 * 20 = 0.264 cycles — a 19% CPI reduction. Reducing L2 miss penalty or increasing L1 hit rates would have a larger absolute impact.

**Common mistake:** Candidates often forget to account for the fetch-width limit in the ideal CPI. A 4-wide superscalar has CPI_ideal = 0.25, not 1.0. The 0.25 baseline means memory stalls and branch mispredictions are amplified relative to a single-issue processor.
