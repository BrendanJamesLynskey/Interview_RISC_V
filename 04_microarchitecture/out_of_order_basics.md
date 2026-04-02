# Out-of-Order Execution Basics

## Prerequisites
- 5-stage in-order RISC-V pipeline and data hazards (RAW/WAR/WAW)
- Forwarding and the load-use hazard
- Branch prediction concepts (speculative execution)

---

## Concept Reference

### Motivation: Why Out-of-Order Execution

In an in-order pipeline, if instruction I3 cannot execute (e.g., it is waiting for a cache miss result), instructions I4, I5, I6, ... must all wait even if they have no dependency on I3. Out-of-order (OoO) execution allows the processor to look ahead and execute independent instructions while waiting for long-latency operations.

```
In-order stall example:
  I1: ADD  x1, x2, x3          -- executes immediately
  I2: LW   x4, 0(x5)           -- L2 cache miss: 50-cycle stall
  I3: SUB  x6, x4, x7          -- depends on I2: must wait
  I4: AND  x8, x9, x10         -- independent of I2/I3: stalls anyway (in-order)
  I5: OR   x11, x12, x13       -- independent: stalls anyway

OoO execution:
  I1 executes
  I2 issued; L2 miss starts; I2 awaits result
  I4, I5 issued and execute while I2 waits (out of program order)
  When I2 completes, I3 can execute
  All results committed in program order (I1, I2, I3, I4, I5)
```

### Tomasulo's Algorithm

Tomasulo's algorithm (IBM 360/91, 1967) is the foundational OoO scheduling algorithm. Modern processors use variants of it.

**Key components:**
1. **Reservation Stations (RS):** Hold instructions and their operands (or tags indicating which instruction will produce a missing operand).
2. **Common Data Bus (CDB):** Broadcasts computed results to all reservation stations simultaneously.
3. **Register Alias Table (RAT) / Register Renaming:** Maps architectural registers to physical registers or reservation station tags.

**Three phases:**

```
Phase 1 — ISSUE (in-order):
  Fetch instruction; assign to a reservation station (RS) slot.
  If RS is full: stall (structural hazard on RS).
  Read operands from register file or rename table.
  If operand is ready (in register file): copy value to RS.
  If operand is not ready (produced by a later instruction still in flight):
      store the tag (RS ID) of the producing instruction in the RS slot.

Phase 2 — EXECUTE (out-of-order):
  When ALL operands in an RS entry are ready (tags resolved to values):
      instruction is eligible to execute.
  Execution unit takes the ready instruction.
  Multiple instructions with ready operands can execute simultaneously on
  different functional units.

Phase 3 — WRITE RESULT (broadcast via CDB):
  Result is broadcast on the CDB with the producer's tag.
  All RS entries that are waiting on this tag capture the value.
  Register file (or RAT) is updated.
```

### Register Renaming

Register renaming eliminates WAR and WAW hazards by giving each instruction write a unique physical register, so no two in-flight instructions share a physical destination register.

```
Without renaming (WAW hazard):
  I1: ADD f1, f2, f3   -- writes architectural register f1 (physical: p1)
  I2: MUL f1, f4, f5   -- also writes f1 (physical: p1) -- WAW hazard

With renaming:
  I1: ADD f1, f2, f3   -- rename f1 -> p10  (free physical register p10 allocated)
  I2: MUL f1, f4, f5   -- rename f1 -> p11  (free physical register p11 allocated)
  I3: ADD f6, f1, f7   -- renames f1 as source -> uses p11 (most recent mapping of f1)

No WAW: p10 and p11 are distinct. I1 and I2 write different physical registers.
No WAR: I3 reads p11 which I2 writes -- clean RAW dependency, handled by forwarding/RS.
```

The mapping from architectural registers to physical registers is maintained in the **Register Alias Table (RAT)**. When an instruction commits from the ROB, the old physical register that was previously mapped to that architectural register can be freed back to the free list.

### Reorder Buffer (ROB)

The ROB is a circular FIFO that maintains program order. All instructions enter the ROB at the ISSUE phase (in program order) and leave (retire/commit) from the head in program order.

```
ROB entry fields:
  PC          : instruction address (for exception reporting)
  instruction : opcode and operands (for re-execution if needed)
  dest_reg    : architectural destination register
  phys_reg    : allocated physical register (result stored here)
  value       : computed result (written when execution completes)
  done        : 1 when execution has completed
  exception   : any pending exception from this instruction
  branch      : branch type, predicted target (for misprediction recovery)

ROB operation:
  Tail: new instructions allocated here (in-order allocation)
  Head: instructions retire here when:
        (a) done == 1
        (b) all preceding instructions have also retired (strict in-order retire)

Retirement:
  Write result to the architectural register file (the "committed" copy).
  Free the OLD physical register that was previously mapped to this arch register.
  If exception: take trap, flush all younger ROB entries.
  If branch misprediction: flush all younger ROB entries; redirect fetch.
```

The ROB provides **precise exceptions**: at any retirement, all instructions with lower PCs have fully completed and their results are committed; no instructions with higher PCs have affected architectural state.

### Tomasulo with ROB — the Modern Variant

The original Tomasulo used the CDB to directly update the register file and other RS entries. Modern processors use the ROB as an intermediate buffer:

```
Classic Tomasulo:           write result directly to register file via CDB
Tomasulo + ROB:             write result to ROB entry; commit to register file at retirement

Advantage of ROB:
  - Enables precise exceptions (results uncommitted until retirement)
  - Enables branch misprediction recovery (flush ROB entries after the branch)
  - Separates speculative from committed state
```

### Issue Queue / Unified vs. Distributed RS

Different processors use different RS organisations:

```
Distributed RS (Tomasulo original):
  Each functional unit has its own RS.
  Advantage: simple; results forwarded directly to local RS.
  Disadvantage: a load/store RS may be full while the ALU RS is empty -- inefficiency.

Unified issue queue (modern designs, e.g., AMD Zen, SiFive P670):
  A single central queue holds all in-flight instructions.
  Any instruction can be dispatched to any available functional unit.
  Better utilisation; more complex arbitration logic.
  Typical size: 32-128 entries.
```

### Load-Store Queue and Memory Ordering

Loads and stores require special handling:
- Stores cannot write to memory until they commit (precise exceptions, speculation).
- Loads may execute out-of-order, but must not return a value that violates memory ordering.

```
Load-Store Queue (LSQ) / Store Queue (SQ) / Load Queue (LQ):

Store Queue:
  Stores are held here from issue until commit.
  On commit: store is released to the L1 data cache.
  Other loads can "snoop" the store queue for load-to-store forwarding:
    if a load and a pending store have the same address, the load can take
    the value from the store queue instead of going to cache.

Load Queue:
  Loads are tracked here until execution completes.
  If a later-committed store updates the same address before the load has been
  fully ordered (memory consistency violation), the load must be re-executed.
  This is detected by the "memory ordering violation" check: comparing
  all pending loads against committed stores.

RISC-V memory ordering:
  RISC-V uses RVWMO (RISC-V Weak Memory Ordering).
  Loads and stores within a single hart may be reordered unless separated by
  FENCE instructions.
  Cross-hart visibility requires careful use of FENCE, FENCE.I, and AMO instructions.
```

### Performance Parameters

```
OoO core key sizing parameters:

ROB size:           typically 128-256 entries (Intel Sunny Cove: 352; AMD Zen 4: 320)
Issue queue:        typically 32-96 entries
Physical registers: architectural count * 3 (roughly); RISC-V I has 32 int regs,
                    so ~96 physical integer registers typical
Functional units:   2-4 ALU units, 1-2 load/store units, 1-2 FPU units
Fetch width:        4-8 instructions/cycle
Retirement width:   4-8 instructions/cycle (must match fetch for steady state)

OoO window:
  The number of instructions the processor can look ahead to find independent work.
  Limited by ROB size and issue queue size.
  Larger window = more instruction-level parallelism (ILP) exposed.
  Diminishing returns above ~256 ROB entries for most workloads.
```

---

## Tier 1 — Fundamentals

### Question F1
**What is the fundamental performance problem that out-of-order execution solves? Give a concrete instruction sequence example.**

**Answer:**

Out-of-order execution solves the problem of **structural stalls blocking independent instructions** in the presence of long-latency operations.

In an in-order pipeline, when instruction I_k stalls (waiting for a cache miss, a long-latency FP divide, or any other multi-cycle operation), every instruction after I_k in program order must also stall, even if those later instructions have no data dependency on I_k. All later instructions sit idle, wasting execution cycles.

**Concrete example — L2 cache miss:**

```
I1: LW   x1, 0(x2)       -- L2 cache miss: must wait 50 cycles
I2: ADD  x3, x1, x4      -- RAW on x1 from I1: must wait for I1
I3: MUL  x5, x6, x7      -- NO dependency on I1 or I2
I4: ADD  x8, x9, x10     -- NO dependency on I1, I2, I3
I5: SLL  x11, x12, x2    -- NO dependency on I1 or I2

In-order behaviour:
  I1 stalls in MEM for 50 cycles.
  I2, I3, I4, I5 all stall behind I1.
  Total extra stall cycles: 50 * 4 = 200 wasted instruction-cycles.

OoO behaviour:
  I1 is issued; begins waiting for L2 response.
  I3, I4, I5 have no dependency on I1: they are issued and execute on available
  functional units during the 50 cycles I1 waits.
  I2 remains pending (it genuinely depends on I1).
  When I1 completes (cycle 50), I2 can immediately execute.
  All 5 instructions commit in program order: I1, I2, I3, I4, I5.

Wasted cycles: only I2 is truly stalled. I3, I4, I5 execute productively.
Effective stall cycles: ~1 (I2 waits for I1 by 1 cycle after I1 completes).
```

The degree of benefit depends on how much independent work is available — the amount of **instruction-level parallelism (ILP)** in the code.

---

### Question F2
**What is register renaming? Which two hazard types (of RAW/WAR/WAW) does it eliminate, and why does it not eliminate the third?**

**Answer:**

Register renaming maps architectural register names (x0-x31 in RISC-V) to a larger set of physical registers. Each time an instruction writes an architectural register, the hardware allocates a fresh physical register for that write, replacing the old mapping in the Register Alias Table (RAT).

**WAR elimination:**
```
Before renaming:
  I1: LW   x1, 0(x2)     -- reads x2 (as base address)
  I2: ADD  x2, x3, x4    -- writes x2 -- WAR: I2 must not write x2 before I1 reads it

After renaming:
  I1: LW   x1, 0(p5)     -- reads physical p5 (previous mapping of x2)
  I2: ADD  p7, p6, p4    -- writes p7 (new mapping of x2); p5 is unchanged
  No conflict: I2 can write p7 at any time without affecting I1's read of p5.
```

**WAW elimination:**
```
Before renaming:
  I1: ADD  x1, x2, x3    -- writes x1
  I2: MUL  x1, x4, x5    -- writes x1 -- WAW: must not overwrite I1's result prematurely

After renaming:
  I1: ADD  p8, p2, p3    -- writes p8
  I2: MUL  p9, p4, p5    -- writes p9
  Both complete independently; no conflict. The most recent mapping of x1 is p9.
```

**Why renaming does not eliminate RAW:**
RAW (Read After Write) is a *true* data dependence. I2 genuinely needs the value that I1 computes. Renaming a register does not create data from nothing — I2 must still wait for I1's result. Renaming changes the physical register I1 writes to (say, p8), and correctly points I2 to read p8. But I2 must still wait until p8 has been written.

```
After renaming:
  I1: ADD  p8, p2, p3    -- I2 must wait for p8 to be written
  I2: SUB  p9, p8, p5    -- depends on p8: genuine RAW, cannot be eliminated
```

RAW hazards are managed by the reservation stations (holding instructions until their operand tags are resolved) and forwarding paths. Renaming converts WAR and WAW into independent writes of distinct physical registers, but it cannot change the fundamental fact that I2 needs I1's computed value.

---

### Question F3
**What is a Reorder Buffer (ROB) and why is it essential for precise exceptions in an OoO processor?**

**Answer:**

The Reorder Buffer (ROB) is a circular FIFO that holds all in-flight instructions in program order. An instruction enters the ROB's tail at the issue/dispatch stage (in program order) and leaves from the ROB's head at the retirement stage — also in program order — after its execution has completed.

**Why it is essential for precise exceptions:**

RISC-V defines a **precise exception model**: when an exception is taken, all instructions with a lower PC have fully completed and updated architectural state; no instruction with a higher PC has yet modified architectural state. This is the "precise" requirement.

In an OoO processor, execution is not in program order. An instruction I5 may complete execution before I2. If I2 subsequently causes a page fault, the processor must roll back to a state where I1 (and only I1) has retired. Without the ROB, I5's result might already be in the register file — impossible to undo cleanly.

**The ROB's role:**

Results are written to the ROB entry (not directly to the architectural register file) when execution completes. The architectural register file is only updated when an instruction **retires from the head of the ROB**. Retirement only occurs in program order, and only when the instruction is at the ROB head and has no preceding uncommitted instructions.

```
Exception handling:
  I1: completes, retires normally (architectural state updated).
  I2: page fault detected when I2 reaches the ROB head.
      mepc = I2's PC.
      All ROB entries at I2 and beyond are flushed.
      Processor state = state after I1 (precise).
      Trap handler invoked.

  I5 (which completed out of order) had its result in the ROB, not in the
  architectural register file. Its ROB entry is flushed; its physical register
  is freed. No architectural corruption occurs.
```

Without the ROB, precise exceptions would require checkpointing all architectural state at every instruction — prohibitively expensive. The ROB provides the required ordering barrier at a cost of ~a few KB of fast SRAM.

---

### Question F4
**Describe the three phases of Tomasulo's algorithm (issue, execute, write result) for the following two-instruction sequence. Show the state of the reservation station after each phase.**

```
I1: ADD f1, f2, f3   (f2=5.0, f3=3.0; result f1=8.0)
I2: MUL f4, f1, f5   (f5=2.0; f1 not yet available)
```

**Answer:**

Assume RS entries: Add1 (for the FP adder), Mult1 (for the FP multiplier). All registers read from the register file; f1 is initially stale (contains some previous value, say 0.0).

**After I1 ISSUE:**
```
RS Add1: op=ADD, Qj=-, Vj=5.0, Qk=-, Vk=3.0  (both operands ready from RF)
RS Mult1: empty

RAT: f1 -> Add1  (f1 will be produced by Add1)
```

**After I2 ISSUE:**
```
RS Add1: op=ADD, Qj=-, Vj=5.0, Qk=-, Vk=3.0  (unchanged; still waiting to execute)
RS Mult1: op=MUL, Qj=Add1, Vj=-, Qk=-, Vk=2.0
           Qj=Add1 means "waiting for the result of reservation station Add1"

RAT: f1 -> Add1  (f1 will be produced by Add1; no change)
     f4 -> Mult1 (f4 will be produced by Mult1)
```

**I1 EXECUTES (FP adder computes 5.0 + 3.0 = 8.0):**
```
FP adder completes; result = 8.0.
RS Add1 result ready.
```

**I1 WRITE RESULT (broadcast on CDB: tag=Add1, value=8.0):**
```
RS Mult1 sees broadcast: Qj == Add1 -> capture value.
RS Mult1: op=MUL, Qj=-, Vj=8.0, Qk=-, Vk=2.0  (both operands now ready)

Register file: f1 <- 8.0 (or ROB entry for I1 updated with 8.0)
RS Add1: freed.
```

**I2 EXECUTES (FP multiplier computes 8.0 * 2.0 = 16.0):**
Mult1 has both operands ready; issues to the FP multiplier.

**I2 WRITE RESULT (broadcast on CDB: tag=Mult1, value=16.0):**
```
Register file: f4 <- 16.0
RS Mult1: freed.
RAT: f4 mapping cleared (f4 now permanently in RF).
```

**Key insight:** I2 could not execute until I1 wrote its result on the CDB. The CDB broadcast is what makes the operand available to I2 without the hardware needing to know, at issue time, when the operand would be ready. This is the elegance of Tomasulo's tag-based forwarding.

---

## Tier 2 — Intermediate

### Question I1
**Consider an OoO processor with a 6-entry ROB and 3 reservation stations per functional unit (one INT unit, one MUL unit, one LOAD unit). For the following instruction stream, trace the ROB and RS state through 5 cycles. Identify any stalls.**

```
I1: LW   x1, 0(x2)    -- cache hit: 2-cycle latency
I2: ADD  x3, x1, x4   -- depends on I1
I3: MUL  x5, x6, x7   -- 3-cycle multiply, independent
I4: ADD  x8, x3, x9   -- depends on I2
I5: LW   x10, 4(x2)   -- independent load, cache hit: 2-cycle latency
```

**Answer:**

**Cycle 1 — Issue I1, I2, I3 (assume 3-wide issue):**
```
ROB:  [1:LW x1] [2:ADD x3] [3:MUL x5]
RS Load1: op=LW, base=x2(ready), imm=0 -> ready to execute
RS INT1:  op=ADD, Qj=ROB1(waiting x1), Vk=x4(ready)
RS MUL1:  op=MUL, Vj=x6(ready), Vk=x7(ready) -> ready to execute
```

**Cycle 2 — Issue I4, I5. Execute I1 (load address sent to cache), I3 (MUL cycle 1):**
```
ROB:  [1:LW x1] [2:ADD x3] [3:MUL x5] [4:ADD x8] [5:LW x10]
RS INT2:  op=ADD, Qj=ROB2(waiting x3), Vk=x9
RS Load2: op=LW, base=x2(ready), imm=4 -> ready to execute

I1: Load cache access in progress (will complete at end of cycle 3)
I3: MUL executing (cycle 1 of 3)
INT1 still waiting for ROB1 (x1 not yet available)
```

**Cycle 3 — Execute: I1 cache returns data, I3 MUL cycle 2, I5 load address sent:**
```
I1: L1 cache hit; x1=result available; broadcasts on CDB: ROB1=value
INT1 sees CDB broadcast: Qj=ROB1 resolved -> INT1 now ready (ADD x3,x1,x4)
I3: MUL cycle 2 of 3
I5: load cache access in progress

ROB1 (LW x1): marked done, value=x1
```

**Cycle 4 — Retire I1 (ROB head, done). Execute: I2 (ADD), I3 MUL cycle 3, I5 cache returns:**
```
I1 retires: x1 written to architectural RF.
I2 (INT1 RS): both operands ready; executes ADD x3=x1+x4; result computed.
I3: MUL cycle 3 of 3 — completes this cycle; result x5 available; CDB broadcast.
I5: L1 cache hit; x10=result; CDB broadcast.

ROB: [2:ADD x3 done] [3:MUL x5 done] [4:ADD x8 waiting ROB2] [5:LW x10 done]
INT2 sees I2 done: Qj=ROB2 -> resolved -> INT2 ready.
```

**Cycle 5 — Retire I2, I3 (both done, in order). Execute I4 (ADD x8):**
```
I2 retires: x3 written to architectural RF.
I3 retires: x5 written to architectural RF.
I4 (INT2): both operands ready; executes ADD x8=x3+x9.
I5: already done.

ROB: [4:ADD x8 executing] [5:LW x10 done]
```

**No structural stalls occurred:** The ROB had capacity (6 entries, only 5 used). The RS units had available slots. The only stalls were data-dependent waits (I2 waiting for I1, I4 waiting for I2), which the OoO engine handled transparently by executing I3 and I5 during those cycles.

---

### Question I2
**Explain the WAR and WAW hazard elimination by register renaming in the context of the following sequence. Show the RAT state after each instruction is issued, using physical registers p0-p15 (architectural registers initially mapped to p0-p7).**

```
Initial: x1->p1, x2->p2, x3->p3, x4->p4, x5->p5, x6->p6
Free list: p8, p9, p10, p11, ...

I1: ADD  x1, x2, x3   -- WAW with I3 (both write x1)
I2: MUL  x4, x1, x5   -- WAR with I3 (reads x1 before I3 can write it)
I3: SUB  x1, x6, x4   -- writes x1 again (WAW with I1); reads x4 (WAR not a problem here)
I4: ADD  x7, x1, x3   -- RAW: reads the x1 written by I3
```

**Answer:**

**After I1 issues (ADD x1, x2, x3):**
```
Rename x1 write -> allocate p8 from free list.
I1 reads: x2->p2 (old mapping), x3->p3 (old mapping)
I1 writes: x1->p8 (new mapping)

RAT: x1->p8, x2->p2, x3->p3, x4->p4, x5->p5, x6->p6
Free list: p9, p10, p11, ...
RS: op=ADD, Vj=p2, Vk=p3, dest=p8
```

**After I2 issues (MUL x4, x1, x5):**
```
Rename x4 write -> allocate p9.
I2 reads: x1->p8 (current mapping, set by I1), x5->p5
I2 writes: x4->p9

RAT: x1->p8, x2->p2, x3->p3, x4->p9, x5->p5, x6->p6
Free list: p10, p11, ...
RS: op=MUL, Qj=p8(waiting I1 result), Vk=p5, dest=p9

No WAR: I2 reads p8 (I1's physical destination). I3 will get a new physical register
for x1. I2's read of p8 is independent of any future writes to x1.
```

**After I3 issues (SUB x1, x6, x4):**
```
Rename x1 write -> allocate p10. (The OLD mapping x1->p8 is noted for later freeing.)
I3 reads: x6->p6 (ready), x4->p9 (I2's physical dest -- waiting for I2)
I3 writes: x1->p10

RAT: x1->p10, x2->p2, x3->p3, x4->p9, x5->p5, x6->p6
Free list: p11, p12, ...
RS: op=SUB, Vj=p6, Qk=p9(waiting I2 result), dest=p10

WAW eliminated: I1 writes p8, I3 writes p10. Distinct physical registers.
                p8 (I1's old write) will be freed when I3 retires (I3's write supersedes I1's).
WAR eliminated: I2 reads p8; I3 writes p10. I3 can complete at any time
                without affecting I2's read of p8.
```

**After I4 issues (ADD x7, x1, x3):**
```
Rename x7 write -> allocate p11.
I4 reads: x1->p10 (I3's physical dest -- waiting for I3), x3->p3

RAT: x1->p10, x2->p2, x3->p3, x4->p9, x5->p5, x6->p6, x7->p11
RS: op=ADD, Qj=p10(waiting I3 result), Vk=p3, dest=p11

RAW remains: I4 genuinely needs I3's result (p10). This dependency cannot be eliminated.
```

**Summary of hazards eliminated:**

| Hazard | Architectural registers | After renaming | Eliminated? |
|--------|------------------------|----------------|-------------|
| WAW I1->I3 | both write x1 | I1 writes p8, I3 writes p10 | Yes |
| WAR I2->I3 | I2 reads x1; I3 writes x1 | I2 reads p8; I3 writes p10 | Yes |
| RAW I1->I2 | I2 reads I1's x1 | I2 reads p8 (I1 writes p8) | No (true dep) |
| RAW I3->I4 | I4 reads I3's x1 | I4 reads p10 (I3 writes p10) | No (true dep) |

---

### Question I3
**Describe memory ordering violations in an OoO processor's load-store queue. What condition triggers a violation, and what is the recovery procedure?**

**Answer:**

A **memory ordering violation** (also called a **load ordering violation** or **memory ordering machine clear**) occurs when a load instruction executes out-of-order and reads a stale value because a store to the same address commits after the load has already executed.

**Scenario:**
```
Program order:
  I1: SW  x1, 0(x2)    -- store to address A
  I2: LW  x3, 0(x2)    -- load from same address A (should read I1's value)

OoO execution (out-of-order):
  I2 executes first (address A, no cache miss) -> reads value V_old from cache.
  I1 executes later -> stores new value V_new to address A.
  I2 has read V_old, but architecturally should have read V_new (from I1).
  Memory ordering violation detected.
```

**Detection mechanism:**

The load queue tracks all pending loads. The store queue tracks all pending stores. When a store commits to the cache:
- The hardware compares the store address against all load queue entries.
- If a load that was younger than this store (in program order) has already executed and read from the same address, a violation is flagged.

```
Load Queue entry: { PC, phys_dest, address, data_read, executed }
Store Queue entry: { PC, address, data, committed }

On store commit to cache:
  for each load in LQ where load.PC > store.PC:   // load is younger
      if load.address == store.address AND load.executed:
          // Violation: this load read a stale value
          // Recovery required
```

**Recovery procedure:**

1. The processor flushes all instructions newer than the violating load from the ROB and execution units.
2. The load is re-executed from its correct program position, now finding the correct store data in the store queue (via load-to-store forwarding) or in the cache.
3. All instructions that depended on the load's (incorrect) result are also re-executed.

This is sometimes called a **memory machine clear** (Intel terminology) or **load re-execution**. It is expensive: approximately 20-40 cycles of wasted work for a deep out-of-order pipeline.

**Mitigation — Load-Store Forwarding:**
If the hardware can predict that a load depends on an in-flight store to the same address, it can forward the store data to the load at issue time, avoiding the violation entirely. This requires address disambiguation: calculating store addresses early enough to compare with incoming load addresses.

**RISC-V memory model note:** RISC-V RVWMO allows a load to read from a store in the same hart's store buffer before the store is globally visible. This is called "store-to-load forwarding" and is architecturally permitted. The violation described above is not a model-permitted reordering; it is a microarchitectural error where a load bypassed an earlier store to the same address without forwarding.

---

## Tier 3 — Advanced

### Question A1
**Compare Tomasulo's original algorithm with the Tomasulo+ROB variant used in modern processors. Specifically address: (a) how register values are communicated to dependent instructions, (b) how WAW hazards are handled, and (c) how exception recovery differs.**

**Answer:**

**Part (a): Communicating register values to dependent instructions**

*Original Tomasulo (no ROB):*
When an instruction writes its result, it broadcasts the value on the Common Data Bus (CDB). Any reservation station entry waiting for this producer's tag immediately receives the value and marks its operand as ready. The register file is also updated via the CDB. The RAT maps architectural registers to RS tags; once the CDB fires, the RAT is updated to point back to the register file.

```
Producer completes -> CDB broadcast (tag, value)
  -> All matching RS entries capture value (operand ready)
  -> Register file updated: arch_reg[dest] = value
  -> RAT clears the RS tag (now pointing to RF again)
```

*Tomasulo + ROB:*
When execution completes, the result is written to the ROB entry (and simultaneously broadcast on the CDB so waiting RS entries can capture it). The architectural register file is NOT updated until the instruction retires from the ROB head. Between execution and retirement, the value lives in the ROB.

```
Producer completes -> CDB broadcast (ROB tag, value)
  -> All matching RS entries capture value
  -> ROB entry for this instruction: result field = value, done = 1
  -> Architectural RF: NOT updated yet

At retirement (ROB head, done == 1):
  -> Architectural RF updated: arch_reg[dest] = value
  -> Old physical reg mapping freed
```

**Part (b): WAW hazard handling**

*Original Tomasulo:*
If I1 and I2 both write f1, and I2 completes first (because it was a short-latency instruction), I2 writes f1 to the register file. When I1 later completes, it also writes f1 — incorrectly overwriting I2's result. Original Tomasulo handles this with a "tag check": I1 only writes f1 if the RAT still shows f1 mapped to I1's RS tag. If I2 has already updated f1 and changed the RAT mapping, I1 skips the register write. This is a subtle piece of logic.

*Tomasulo + ROB:*
WAW is handled cleanly by retirement order. I2 always retires after I1 (ROB enforces program order). When I1 retires, it writes f1. When I2 retires, it overwrites f1 with the correct (newer) value. In practice, the physical register renaming removes WAW entirely at the architectural register level — I1 and I2 write to different physical registers, and only the most recent mapping (I2's) is used by subsequent instructions. The old physical register (I1's) is freed when I2 retires.

**Part (c): Exception recovery**

*Original Tomasulo:*
No ROB means no way to restore precise state. If I3 causes an exception but I5 (which executed out of order before I3) has already written its result to the register file, the processor has no record of what f5 contained before I5 executed. Restoring to the pre-I3 state is impossible without explicit checkpointing. **Original Tomasulo does not support precise exceptions.** This is a major limitation that made it unsuitable for software environments requiring precise traps.

*Tomasulo + ROB:*
The ROB provides precise exceptions naturally. Results are buffered in the ROB until retirement. If I3 causes a page fault when it reaches the ROB head:
1. All ROB entries from I3 onward are flushed.
2. Their physical registers are freed.
3. The architectural register file reflects the state after all instructions preceding I3 — exactly the RISC-V precise exception requirement.
4. `mepc` is set to I3's PC; `mcause` to the exception cause.
5. The trap handler executes with a consistent architectural state.

---

### Question A2
**A RISC-V OoO processor uses a unified physical register file with 192 entries for the 32 integer architectural registers. Describe the complete lifecycle of a physical register from allocation to freeing, and explain the "age-based" freeing problem that arises when the free list is exhausted.**

**Answer:**

**Physical register lifecycle:**

```
Step 1 — RENAME (at issue/dispatch):
  New instruction writes arch_reg Ra.
  Old physical register mapping: Ra -> p_old (from RAT).
  Allocate new physical register p_new from free list.
  Update RAT: Ra -> p_new.
  Save p_old in the ROB entry for this instruction (needed at retirement).

Step 2 — EXECUTE:
  Instruction computes result; writes result to p_new in the physical RF.

Step 3 — WRITE RESULT:
  p_new's valid bit is set; value is stable.
  CDB broadcasts (ROB_tag, value) to waiting RS entries.

Step 4 — RETIRE (at ROB head):
  The architectural register file's "committed" mapping of Ra is updated to p_new.
  p_old is now stale: no committed instruction references it.
  p_old is returned to the free list.
```

**Why p_old cannot be freed earlier than retirement:**

Before this instruction retires, there might be a branch misprediction or an exception that requires rolling back to a state where Ra still maps to p_old. If p_old were freed (and potentially reallocated) before retirement, the rollback would have no valid data to restore.

```
Timeline:
  Issue:   Ra -> p_old; allocate p_new; ROB entry saves p_old.
  Execute: p_new receives result.
  Retire:  Ra -> p_new committed; p_old freed.

If branch misprediction before retirement:
  Flush ROB entries. Rollback RAT to the pre-issue state.
  Ra -> p_old restored (valid: p_old was never freed).
  p_new freed (it was speculative).
```

**The free list exhaustion ("register pressure") problem:**

With 192 physical registers and 32 architectural registers, there are 160 "extra" physical registers available for in-flight instructions. Each instruction that writes a register consumes one physical register from the free list (until it retires and frees the old one). If the processor issues instructions faster than it retires them, the free list can be exhausted.

**Example:**
```
192 physical regs, 32 committed mappings => 160 available.
Each in-flight write instruction holds 1 physical reg until it retires.
If the pipeline stalls at retirement (e.g., waiting for a long-latency L3 miss),
in-flight instructions accumulate. With 160 available regs, the pipeline can
tolerate up to 160 in-flight write instructions before stalling.

In practice: ROB is typically sized smaller than the physical reg file minus the
             architectural register count, so the ROB fills before the free list
             exhausts. With ROB=128 and free_regs=160, ROB is the limiting factor.
```

**"Age-based" (oldest-first) freeing:**

When the free list runs out and the pipeline must stall, the processor cannot simply free the physical register mapped to the architectural register file — those may still be needed by in-flight instructions. The stall must persist until the oldest ROB entry retires (freeing one physical register), then the next-oldest, etc. This is an age-ordered serialisation of retirements.

The fix is correct sizing: ensure `|physical_regs| - 32 > ROB_size` so the ROB fills before the physical register file exhausts. Alternatively, use **register file checkpointing** (snapshot the RAT at branch points) to enable faster recovery without waiting for age-ordered retirement — used in some high-IPC designs.

---

### Question A3
**Design a simplified ROB in SystemVerilog (parameterised, 8-entry for clarity). Show the fields, allocation (issue), completion (write result), and retirement (commit) logic. Discuss how this scales to a 256-entry production ROB.**

**Answer:**

```systemverilog
// Simplified 8-entry ROB for a scalar in-order issue, OoO execute pipeline
// (Instruction issue is in-order; execution and retirement use the ROB)

module reorder_buffer #(
    parameter ROB_DEPTH  = 8,
    parameter ROB_PTR_W  = 3,   // log2(ROB_DEPTH)
    parameter DATA_W     = 32,
    parameter REG_W      = 5    // 5-bit RISC-V register indices
) (
    input  logic                clk,
    input  logic                rst_n,

    // ---- ISSUE interface (in-order allocation at tail) ----
    input  logic                issue_valid,    // new instruction being issued
    input  logic [REG_W-1:0]    issue_dest_reg, // architectural destination register
    input  logic [DATA_W-1:0]   issue_pc,       // for exception reporting
    output logic [ROB_PTR_W-1:0] issue_rob_tag, // ROB slot assigned to this instruction
    output logic                issue_ready,    // ROB has space (not full)

    // ---- COMPLETE interface (out-of-order result write) ----
    input  logic                complete_valid,
    input  logic [ROB_PTR_W-1:0] complete_tag,  // which ROB entry completed
    input  logic [DATA_W-1:0]   complete_data,  // result value
    input  logic                complete_exception, // did instruction fault?

    // ---- RETIRE interface (in-order commit at head) ----
    output logic                retire_valid,   // head of ROB is done: ready to retire
    output logic [REG_W-1:0]    retire_dest_reg,
    output logic [DATA_W-1:0]   retire_data,
    output logic [DATA_W-1:0]   retire_pc,
    output logic                retire_exception,
    input  logic                retire_ack      // external logic acknowledges retirement
);

    // ROB entry definition
    typedef struct packed {
        logic [DATA_W-1:0]  pc;
        logic [REG_W-1:0]   dest_reg;
        logic [DATA_W-1:0]  result;
        logic               done;
        logic               exception;
        logic               valid;       // entry is occupied
    } rob_entry_t;

    rob_entry_t rob [ROB_DEPTH-1:0];

    logic [ROB_PTR_W-1:0] head;    // oldest entry (next to retire)
    logic [ROB_PTR_W-1:0] tail;    // next free slot
    logic [ROB_PTR_W:0]   count;   // number of occupied entries (one extra bit)

    logic full  = (count == ROB_DEPTH);
    logic empty = (count == 0);

    assign issue_ready   = ~full;
    assign issue_rob_tag = tail;

    // Retirement outputs from ROB head
    assign retire_valid     = ~empty && rob[head].valid && rob[head].done;
    assign retire_dest_reg  = rob[head].dest_reg;
    assign retire_data      = rob[head].result;
    assign retire_pc        = rob[head].pc;
    assign retire_exception = rob[head].exception;

    always_ff @(posedge clk or negedge rst_n) begin
        if (~rst_n) begin
            head  <= '0;
            tail  <= '0;
            count <= '0;
            for (int i = 0; i < ROB_DEPTH; i++) rob[i] <= '0;
        end else begin

            // ISSUE: allocate tail slot
            if (issue_valid && issue_ready) begin
                rob[tail].pc        <= issue_pc;
                rob[tail].dest_reg  <= issue_dest_reg;
                rob[tail].result    <= '0;
                rob[tail].done      <= 1'b0;
                rob[tail].exception <= 1'b0;
                rob[tail].valid     <= 1'b1;
                tail  <= tail + 1;
                count <= count + 1;
            end

            // COMPLETE: write result to the specified ROB entry
            if (complete_valid) begin
                rob[complete_tag].result    <= complete_data;
                rob[complete_tag].exception <= complete_exception;
                rob[complete_tag].done      <= 1'b1;
            end

            // RETIRE: commit head entry when done and acknowledged
            if (retire_valid && retire_ack) begin
                rob[head].valid <= 1'b0;
                head  <= head + 1;
                count <= count - (issue_valid && issue_ready ? 0 : 1);
                // Simplified: count decrement without simultaneous issue correction
                // (production code handles the combined case carefully)
            end

        end
    end

endmodule
```

**Scaling to a 256-entry production ROB:**

1. **Width scaling:** Each ROB entry must hold the PC (64-bit for RV64), destination register index, result value (64-bit for FP/int), a valid/done/exception flag, and the old physical register tag for renaming. At 256 entries, total storage = 256 * (64+5+64+3+5) bits = 256 * 141 bits = ~4.5 Kbytes of SRAM, well within budget.

2. **Multi-port access:** A production core issues 4-8 instructions per cycle (4-8 writes to the ROB tail) and retires 4-8 per cycle (4-8 reads from the ROB head). The ROB must support multi-port access. This is implemented with multi-bank SRAM (e.g., 4 banks, one port each, with interleaving to provide 4 independent access ports).

3. **Completion port conflicts:** In a wide OoO core, multiple functional units may complete in the same cycle. Each needs to write its ROB entry. With 4 ALU units + 2 load units + 2 FP units = 8 possible completions per cycle, the ROB needs 8 write ports. This is typically handled with a priority arbiter that queues completions, since simultaneous writes to distinct entries are structurally independent.

4. **Recovery (flush) logic:** On a branch misprediction, all ROB entries from the branch to the tail must be invalidated in one cycle (or a small number of cycles). With 256 entries, a 256-bit valid mask and a single-cycle OR-reduction to the new tail pointer is feasible. Physical register freeing during flush requires broadcasting all freed physical reg indices to the free list in the same cycle.

5. **Power:** At 4 GHz with a 4-wide machine, the ROB is accessed roughly 16 billion times per second (4 issues + 4 retirements + N completions per cycle). SRAM power is proportional to access frequency. Techniques: clock gating unused entries, using register files with read suppression, and keeping the ROB on the critical power budget.
