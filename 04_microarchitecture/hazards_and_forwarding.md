# Hazards and Forwarding

## Prerequisites
- The classic 5-stage RISC-V pipeline (IF, ID, EX, MEM, WB) and pipeline registers
- RISC-V RV32I instruction encoding: R-, I-, S-, B-type formats
- Register file structure: two read ports, one write port

---

## Concept Reference

### The Three Kinds of Data Hazard

A **data hazard** occurs when an instruction needs a value that has not yet been produced by a preceding instruction still in the pipeline. There are three types, defined by the ordering of the read (R) and write (W) events:

```
Instruction sequence:   I1 followed by I2

RAW — Read After Write (true dependence):
  I1 writes register Rx; I2 reads register Rx before I1's write completes.
  This is the only hazard that occurs in a classic in-order 5-stage pipeline.
  Example:
    ADD x1, x2, x3   -- writes x1 in WB (cycle 5)
    SUB x4, x1, x5   -- reads  x1 in ID (cycle 3) -- reads STALE value

WAR — Write After Read (anti-dependence):
  I2 writes Rx before I1 has read Rx.
  Cannot occur in a simple in-order pipeline because I1 always reads Rx in ID
  before I2 can possibly write it (I2 enters the pipeline after I1).
  Relevant in out-of-order execution or with register renaming.
  Example:
    LW  x1, 0(x2)    -- reads  x2 in EX (uses x2 as base)
    ADD x2, x3, x4   -- writes x2 in WB; fine because LW already read x2

WAW — Write After Write (output dependence):
  Two instructions both write the same register; the later write must not be
  lost or reordered.
  Cannot occur in a simple in-order pipeline because writes always complete
  in program order.
  Relevant in out-of-order execution.
  Example:
    ADD x1, x2, x3   -- writes x1
    MUL x1, x4, x5   -- also writes x1; the final value must come from MUL
```

**Key point for interviews:** In a standard 5-stage in-order pipeline, only RAW hazards cause problems. WAR and WAW are discussed in the context of out-of-order execution.

### RAW Hazard Timeline

Without forwarding, a RAW hazard requires the pipeline to stall until the producing instruction completes WB and the value is in the register file:

```
Cycle:       1    2    3    4    5    6    7    8    9
ADD x1,...: IF   ID   EX  MEM   WB
            --------- writes x1 here ---------------^
SUB ...,x1:      IF   ID  [stall][stall][stall] EX  MEM  WB
                      ^reads x1 -- needs to wait 3 cycles without forwarding
```

Three stall cycles are required per RAW hazard without any forwarding. This is unacceptable for a real processor.

### Forwarding (Bypassing)

Forwarding routes a result from the pipeline register where it currently resides directly to the ALU input that needs it, skipping the register file entirely.

```
5-stage pipeline with forwarding paths:

         IF     ID     EX    MEM    WB
          |      |      |      |      |
       IF/ID  ID/EX  EX/MEM MEM/WB   |
                |      |      |      |
                +------+------+------+-- register file write
                       |      |
                  ForwardA  ForwardB   (ALU input muxes)
                       |      |
                    ALU input A, B

Two forwarding sources:
  EX/MEM.ALUresult  -> "EX-to-EX forwarding" (1-cycle-old result)
  MEM/WB.result     -> "MEM-to-EX forwarding" (2-cycle-old result)
```

With both forwarding paths, most RAW hazards are resolved with zero stall cycles. The one exception is the load-use hazard (see below).

### The Complete Forwarding Conditions

Let the current instruction entering EX be the "consumer" and define:

```
Consumer:   ID/EX.rs1, ID/EX.rs2  (source register indices)
Producer 1: EX/MEM.rd, EX/MEM.RegWrite  (instruction one stage ahead)
Producer 2: MEM/WB.rd, MEM/WB.RegWrite  (instruction two stages ahead)

Forward to ALU input A:
  if (EX/MEM.RegWrite == 1) AND (EX/MEM.rd != x0) AND (EX/MEM.rd == ID/EX.rs1):
      ForwardA = EX_MEM   -- EX/MEM.ALUresult -> ALU input A

  else if (MEM/WB.RegWrite == 1) AND (MEM/WB.rd != x0) AND (MEM/WB.rd == ID/EX.rs1):
      ForwardA = MEM_WB   -- MEM/WB result -> ALU input A

  else:
      ForwardA = REG      -- register file output (no hazard)

Forward to ALU input B (symmetric, replace rs1 with rs2):
  if (EX/MEM.RegWrite == 1) AND (EX/MEM.rd != x0) AND (EX/MEM.rd == ID/EX.rs2):
      ForwardB = EX_MEM

  else if (MEM/WB.RegWrite == 1) AND (MEM/WB.rd != x0) AND (MEM/WB.rd == ID/EX.rs2):
      ForwardB = MEM_WB

  else:
      ForwardB = REG
```

**Why check rd != x0:** Register x0 is hardwired to zero. An instruction that "writes" x0 (e.g., SW has rd=x0 for encoding purposes; or genuinely writing x0) must never forward its result to a consumer, because the consumer expects x0 = 0, not some computed value.

**Why EX/MEM takes priority over MEM/WB:** If two consecutive instructions both write the same register and a third instruction reads it, the most recent write (EX/MEM) contains the correct final value. The older write (MEM/WB) would be a stale overwrite.

### The Load-Use Hazard

The load-use hazard is the one case where forwarding cannot eliminate the stall:

```
Cycle:      1    2    3    4    5    6    7
LW x1,...: IF   ID   EX  MEM   WB
                          ^-- memory read data available (end of MEM, cycle 4)
ADD ..,x1:      IF   ID   EX  MEM   WB
                      ^-- needs x1 here (start of EX, cycle 3 -- one cycle too early)
```

The LW data is produced at the end of the MEM stage. The dependent instruction needs it at the beginning of its EX stage. The data is needed one cycle before it is available. No forwarding path can travel backward in time.

**Hardware stall mechanism (hazard detection unit):**

```
Load-use detection (in ID stage, examining ID/EX register for the preceding instr):

  if (ID/EX.MemRead == 1) AND
     (ID/EX.rd == IF/ID.rs1  OR  ID/EX.rd == IF/ID.rs2):
       STALL

Actions on STALL:
  1. Freeze the PC register (do not increment PC)
  2. Freeze the IF/ID pipeline register (re-present the same instruction)
  3. Insert a NOP bubble into ID/EX (zero out RegWrite, MemRead, MemWrite)
```

After the one-cycle stall, the LW result sits in EX/MEM and can be forwarded normally.

```
With stall and forwarding:

Cycle:      1    2    3    4    5    6    7
LW x1,...: IF   ID   EX  MEM   WB
ADD ..,x1:      IF   ID [stall] EX  MEM   WB
                               ^-- EX/MEM.MemData forwarded here
```

### Store-Load Forwarding

When a store is followed immediately by a load to the same address, the data must be forwarded from the store data path to the load result path. This is handled differently depending on whether the store value has already left the pipeline:

```
SW x3, 0(x1)    -- x3 value in EX/MEM.rs2_data
LW x4, 0(x1)    -- loads from same address
```

If both hit the same cache set simultaneously, the data cache or the store buffer must forward the store data to the load. This is a **memory-level RAW** hazard, separate from register forwarding.

---

## Tier 1 — Fundamentals

### Question F1
**Define the three types of data hazard (RAW, WAR, WAW). Which type causes stalls in a classic 5-stage in-order RISC-V pipeline, and why do the other two not?**

**Answer:**

A data hazard arises when an instruction needs a value, or needs to write a value, at a time when the pipeline's ordering does not naturally provide the correct result.

**RAW (Read After Write) — true dependence:**
Instruction I2 reads a register that I1 has not yet written. This is called a "true" dependence because the data flow is genuine: I2 really does need I1's result. RAW is the only hazard that matters in a simple in-order 5-stage pipeline.

**WAR (Write After Read) — anti-dependence:**
Instruction I2 writes a register that I1 has not yet read. In a 5-stage in-order pipeline, I1 always enters the pipeline before I2. I1 reads its source registers in the ID stage; I2 writes its destination in WB. Since I1's ID stage always completes before I2's WB stage, I1 will always have read the old value before I2 overwrites it. WAR hazards cannot cause incorrect results in in-order pipelines.

**WAW (Write After Write) — output dependence:**
Instructions I1 and I2 both write the same register; the final value must be I2's, since it is the later instruction. In an in-order pipeline, WB stages happen in program order: I1's WB always precedes I2's WB. The correct final value is always written last. WAW hazards cannot cause incorrect results in in-order pipelines.

**Conclusion:** Only RAW hazards require hardware intervention (forwarding or stalls) in a simple in-order pipeline. WAR and WAW become relevant in out-of-order processors where instructions can complete out of program order.

---

### Question F2
**What is forwarding (bypassing) and why does it improve performance? Give a concrete example with a two-instruction sequence.**

**Answer:**

Forwarding is a hardware technique that routes a computed result directly from the pipeline register where it currently resides to the ALU input that needs it, without waiting for the result to be written to and read back from the register file.

**Without forwarding:**
```
ADD x1, x2, x3    -- writes x1 in WB (cycle 5 if it starts in cycle 1)
SUB x4, x1, x5    -- reads x1 in ID (cycle 3) -- register file has stale value

Without forwarding, SUB must stall for 3 cycles until ADD completes WB.
CPI cost: +3 stall cycles per RAW-dependent pair.
```

**With forwarding:**
```
Cycle:       1    2    3    4    5
ADD x1,...: IF   ID   EX  MEM   WB
                        ^-- ALU result for x1 available here (end of EX, cycle 3)

SUB ..,x1,:      IF   ID   EX  MEM   WB
                            ^-- needs x1 here (start of EX, cycle 3)

Forward path: EX/MEM.ALUresult -> SUB's ALU input A
No stall required.
```

The forwarding MUX at the ALU input selects between:
- The register file output (no hazard)
- EX/MEM.ALUresult (1-cycle-old result)
- MEM/WB.result (2-cycle-old result, covers the case where two instructions separate the producer and consumer)

**Performance impact:** Without forwarding, every RAW-dependent pair costs 3 stall cycles. With full forwarding, zero stall cycles are needed for register-ALU RAW hazards. In typical RISC-V code where 25-40% of instructions have a data dependency on one of the two immediately preceding instructions, forwarding can eliminate the majority of stall cycles.

---

### Question F3
**What is the load-use hazard? Why can forwarding not resolve it? What does the hardware do instead?**

**Answer:**

The load-use hazard occurs when a load instruction (LW, LH, LB, etc.) is immediately followed by an instruction that uses the loaded value:

```
LW  x1, 0(x2)    -- x1 = memory[x2]
ADD x3, x1, x4   -- uses x1 immediately
```

**Why forwarding cannot help:** The LW instruction does not produce its result until the end of the MEM stage (when the data cache returns the read data). In the pipeline diagram:

```
Cycle:      1    2    3    4    5
LW x1:     IF   ID   EX  MEM   WB
                          ^-- x1 available here (end of cycle 4)
ADD ..,x1:      IF   ID   EX  MEM  WB
                       ^-- needs x1 here (start of cycle 4 = end of cycle 3)
```

ADD needs x1 one full cycle before LW can provide it. Forwarding wires move data spatially through the chip; they cannot move it backward in time.

**Hardware solution — hazard detection and stall:**

The hazard detection unit monitors the ID/EX pipeline register. When it sees that the instruction in EX is a load AND its destination register matches a source register of the instruction currently in ID, it takes three actions:

1. Holds the PC register fixed (so the same instruction address is re-presented to IF next cycle).
2. Holds the IF/ID pipeline register fixed (so the same instruction is re-decoded).
3. Injects a NOP bubble into ID/EX (all control signals set to zero, preventing any state change).

This effectively inserts one idle cycle (a "stall" or "pipeline bubble"). After the stall, the LW result is in EX/MEM and can be forwarded normally to ADD's ALU input.

**CPI cost:** Every load-use pair adds exactly 1 stall cycle. Compilers can often eliminate this by reordering instructions (placing an independent instruction between the load and its consumer) — this is called **load scheduling**.

---

### Question F4
**Explain the role of the x0 register in forwarding logic. Why must the forwarding conditions explicitly exclude rd == x0?**

**Answer:**

In RISC-V, register x0 is hardwired to zero. Any instruction that "writes" x0 has no effect; the value 0 is always returned regardless of what was written.

If the forwarding logic did not exclude rd == x0, the following bug could occur:

```
SW x5, 0(x3)    -- Store: instruction encoding has rd=x0 (S-type format has no rd field,
                            but the pipeline register may propagate rd=0 from the encoding)
                            RegWrite is 0 for stores, but rd field reads as x0.
ADD x7, x0, x4 -- uses x0; should read 0

If RegWrite == 0 is the only guard and rd comparison is not checked separately,
a store might incorrectly appear to "forward" a garbage ALU result to x0.
```

More concretely: some instructions produce a value in the ALU result pipeline register (e.g., SW computes the store address in EX) but have RegWrite=0. If a subsequent instruction reads x0 and the forwarding logic only checks RegWrite (not rd != x0), a subtle implementation may still forward a non-zero value.

**Correct forwarding condition always includes both guards:**

```
if (EX/MEM.RegWrite == 1) AND (EX/MEM.rd != x0) AND (EX/MEM.rd == consumer.rs1):
    forward
```

The `rd != x0` check ensures that no instruction that writes x0 (which has no architectural effect) triggers a forward, since the consumer should receive 0 from the register file, not some transient pipeline value.

---

## Tier 2 — Intermediate

### Question I1
**For the following RISC-V instruction sequence, identify every data hazard (type and which registers are involved), determine which are resolved by forwarding, and identify any that require a stall. Draw the pipeline diagram showing forwarding paths.**

```
I1: ADD  x1, x2, x3    # x1 = x2 + x3
I2: SUB  x4, x1, x5    # x4 = x1 - x5  (uses x1 from I1)
I3: LW   x6, 0(x4)     # x6 = mem[x4]  (uses x4 from I2)
I4: AND  x7, x6, x1    # x7 = x6 & x1  (uses x6 from I3, x1 from I1)
I5: OR   x8, x7, x4    # x8 = x7 | x4  (uses x7 from I4)
```

**Answer:**

**Hazard identification:**

| Producer | Consumer | Register | Type | Distance | Resolution         |
|----------|----------|----------|------|-----------|--------------------|
| I1 ADD   | I2 SUB   | x1       | RAW  | 1 cycle   | EX/MEM forward (ForwardA) |
| I1 ADD   | I4 AND   | x1       | RAW  | 3 cycles  | MEM/WB forward (ForwardB) |
| I2 SUB   | I3 LW    | x4       | RAW  | 1 cycle   | EX/MEM forward (ForwardA for base address) |
| I3 LW    | I4 AND   | x6       | RAW  | 1 cycle   | LOAD-USE: requires 1 stall + EX/MEM forward |
| I4 AND   | I5 OR    | x7       | RAW  | 1 cycle   | EX/MEM forward |

**Pipeline diagram (with stall on I3->I4 load-use):**

```
Cycle:  1    2    3    4    5    6    7    8    9   10
I1:    IF   ID   EX  MEM   WB
I2:         IF   ID   EX  MEM   WB
I3:              IF   ID   EX  MEM   WB
I4:                   IF   ID [stall] EX  MEM   WB
I5:                        IF [stall] ID   EX  MEM   WB
                                      ^
                              Hazard detection stalls I4/I5

Forwarding paths:
  Cycle 4: I1 in MEM, I2 in EX   -> EX/MEM.ALUresult (x1) forwarded to I2 ALU input A
  Cycle 5: I2 in MEM, I3 in EX   -> EX/MEM.ALUresult (x4) forwarded to I3 ALU input A (base)
  Cycle 5: I1 in WB,  I4 in ID   -> x1 forwarded (but I4 is stalled in ID, so no action yet)
  Cycle 6: I3 in MEM, I4 in EX   -> EX/MEM.MemData (x6) forwarded to I4 ALU input A
  Cycle 6: I2 in WB,  I4 in EX   -> MEM/WB.ALUresult (x4) forwarded to I4 ALU input B? 
             No -- I4 uses x6 and x1, not x4. Check: MEM/WB.rd=x4, I4.rs2=x1 -> no match.
             I1 result (x1): I1 is past WB by cycle 6; already in register file.
  Cycle 7: I4 in MEM, I5 in EX   -> EX/MEM.ALUresult (x7) forwarded to I5 ALU input A
```

**Key observation for I4's x1 dependency:** I1 writes x1; I4 is 3 instructions later but is delayed by 1 stall cycle (so it is effectively 4 cycles behind I1). By the time I4 reaches EX (cycle 7), I1 has completed WB (cycle 5). The value is in the register file and read normally in ID — no forwarding needed.

**Total stall cycles:** 1 (from the LW-AND load-use hazard).

---

### Question I2
**Design the forwarding MUX select logic (in RTL pseudocode) for ALU input A of a 5-stage RISC-V pipeline. Include the double-hazard priority case.**

**Answer:**

The double-hazard case: two consecutive instructions both write the same destination register, and the next instruction reads it. The EX/MEM register holds the more recent (correct) value; MEM/WB holds the older (stale) value. EX/MEM must take priority.

```verilog
// Pipeline register fields assumed visible to the forwarding unit:
//   EX_MEM_RegWrite   : 1-bit
//   EX_MEM_rd         : 5-bit destination register index
//   MEM_WB_RegWrite   : 1-bit
//   MEM_WB_rd         : 5-bit destination register index
//   ID_EX_rs1         : 5-bit source register 1 index for instruction in EX
//
// ForwardA encoding:
//   2'b00 = use register file output (ID/EX.rs1_data)
//   2'b10 = use EX/MEM.ALU_result   (EX-to-EX forward)
//   2'b01 = use MEM/WB.result       (MEM-to-EX forward)

always_comb begin
    // Default: no forwarding
    ForwardA = 2'b00;

    // Check MEM/WB first (lower priority); will be overridden if EX/MEM also matches
    if (MEM_WB_RegWrite &&
        (MEM_WB_rd != 5'b0) &&
        (MEM_WB_rd == ID_EX_rs1)) begin
        ForwardA = 2'b01;
    end

    // Check EX/MEM second (higher priority); overrides MEM/WB match
    if (EX_MEM_RegWrite &&
        (EX_MEM_rd != 5'b0) &&
        (EX_MEM_rd == ID_EX_rs1)) begin
        ForwardA = 2'b10;   // EX/MEM has the freshest value
    end
end
```

**Why assign MEM/WB first then override with EX/MEM:** This priority order ensures that in the double-hazard case (where both EX/MEM and MEM/WB match rs1), EX/MEM wins. An equivalent formulation uses explicit priority:

```verilog
always_comb begin
    if (EX_MEM_RegWrite && (EX_MEM_rd != 5'b0) && (EX_MEM_rd == ID_EX_rs1))
        ForwardA = 2'b10;
    else if (MEM_WB_RegWrite && (MEM_WB_rd != 5'b0) && (MEM_WB_rd == ID_EX_rs1))
        ForwardA = 2'b01;
    else
        ForwardA = 2'b00;
end
```

**Symmetric logic applies for ForwardB**, replacing `ID_EX_rs1` with `ID_EX_rs2`.

**The complete ALU input A MUX:**
```verilog
always_comb begin
    case (ForwardA)
        2'b00:  alu_input_a = ID_EX_rs1_data;     // from register file
        2'b01:  alu_input_a = MEM_WB_result;       // from MEM/WB (load data or ALU)
        2'b10:  alu_input_a = EX_MEM_ALU_result;   // from EX/MEM (most recent ALU)
        default: alu_input_a = ID_EX_rs1_data;
    endcase
end
```

---

### Question I3
**A RISC-V processor supports the V (vector) extension. Explain how vector instructions complicate the forwarding and hazard detection logic compared to scalar instructions.**

**Answer:**

Vector instructions introduce several complications:

**1. Variable execution latency:**
Scalar ALU instructions complete in one clock cycle in the EX stage. Vector instructions operate on up to VLEN/SEW elements and may take multiple cycles to complete. For example, a VADD.VV with VLEN=256 and SEW=32 processes 8 elements; the execution unit may be pipelined internally but takes several cycles to produce the full result vector.

```
Scalar:  ADD x1, x2, x3   -- result in EX/MEM after 1 cycle in EX
Vector:  VADD.VV v1,v2,v3 -- result may take 4+ cycles in EX unit
```

A simple 1-cycle or 2-cycle forwarding path is insufficient. The hazard detection unit must track the completion cycle of every in-flight vector instruction and stall dependent instructions accordingly.

**2. WAW hazards become possible with scalar/vector mixing:**
If a vector instruction with a long latency is followed by a shorter scalar instruction writing the same vector register group, WAW can occur. Even in an in-order pipeline, the scalar instruction may retire first.

**3. vl (vector length) register dependency:**
Many vector instructions depend on the value of the `vl` (vector length) CSR, set by `vsetvli`. If `vsetvli` is immediately followed by a vector instruction, a RAW hazard exists on the vl register. This is a CSR-level data hazard not present in scalar pipelines.

**4. Mask register dependencies:**
When vector mask registers (v0) are used as predicates (e.g., `VADD.VV v1, v2, v3, v0.t`), the pipeline must track RAW hazards on v0 just as it does on scalar registers, but v0 spans VLEN bits.

**5. Forwarding complexity:**
Forwarding a full VLEN-bit vector (e.g., 512 bits for RVV) requires wide forwarding buses across the chip. At high clock speeds this becomes a routing and timing challenge, motivating designs that use scoreboards or interlocks rather than full forwarding for vector operands.

---

## Tier 3 — Advanced

### Question A1
**WAW hazards are described as impossible in a simple 5-stage in-order pipeline. Precisely describe a realistic RISC-V microarchitecture configuration where WAW hazards CAN occur in an otherwise in-order core, and explain the detection and resolution mechanism.**

**Answer:**

WAW hazards can occur in an in-order core when the pipeline has **non-uniform execution latencies** — specifically when a floating-point or multiply unit has a longer pipeline than the integer unit.

**Scenario: Dual-execution unit with FP unit (3-cycle latency) and integer unit (1-cycle latency):**

```
Instruction stream:
  I1: FADD.S f1, f2, f3   -- FP add, 3 cycles in EX unit, writes f1
  I2: FMUL.S f1, f4, f5   -- FP mul, 3 cycles in EX unit, writes f1  (WAW with I1)
  I3: FADD.S f6, f1, f7   -- reads f1 (RAW with I2, not I1)

Timeline (assuming in-order issue but variable-latency FP unit):

  Cycle:  1    2    3    4    5    6    7
  I1:    IF   ID   EX1  EX2  EX3  WB
  I2:         IF   ID   EX1  EX2  EX3  WB
  I3:              IF   ID   stall stall EX  ...
```

I1 completes WB in cycle 6. I2 completes WB in cycle 7. The WB stage has only one write port; both I1 and I2 want to write f1, with I1 attempting it one cycle before I2. This is a WAW hazard: I1's (incorrect, stale for I3) write must not be lost, and I3 must receive I2's result.

**Detection:**
The write-back stage needs a structural conflict detector that checks whether two instructions will complete WB in the same cycle and write the same register. More commonly, the in-order core checks for WAW at issue time: if instruction I2 writes a register that I1 (still in the long-latency unit) will also write, the processor can:

1. **Squash I1's WB:** The later instruction I2's write takes priority; I1's result is simply discarded when I2 catches up.
2. **Stall I2 at issue:** Prevent I2 from entering the FP unit until I1 has completed WB, eliminating the conflict.

**RISC-V relevance:**
The RISC-V privileged specification and the FP extension do not mandate specific pipeline depths, so WAW management is implementation-defined. Processors like the SiFive U74 and Western Digital SweRV EH1 handle this through a scoreboard that tracks all in-flight writes and stalls conflicting issues.

---

### Question A2
**Describe the complete hardware implementation of the load-use hazard detection and stall insertion mechanism in SystemVerilog. Include the PC freeze, IF/ID freeze, and ID/EX flush logic.**

**Answer:**

```systemverilog
module hazard_detection_unit (
    // From ID/EX pipeline register (the instruction in EX stage)
    input  logic        id_ex_mem_read,   // 1 if instruction in EX is a load
    input  logic [4:0]  id_ex_rd,         // destination register of instr in EX

    // From IF/ID pipeline register (the instruction in ID stage -- potential consumer)
    input  logic [4:0]  if_id_rs1,        // source register 1 of instr in ID
    input  logic [4:0]  if_id_rs2,        // source register 2 of instr in ID

    // Stall control outputs
    output logic        pc_write,         // 0 = freeze PC, 1 = allow PC update
    output logic        if_id_write,      // 0 = freeze IF/ID register, 1 = allow update
    output logic        id_ex_flush       // 1 = inject NOP bubble into ID/EX
);

    logic load_use_hazard;

    always_comb begin
        load_use_hazard = id_ex_mem_read &&
                          (id_ex_rd != 5'b0) &&
                          ((id_ex_rd == if_id_rs1) || (id_ex_rd == if_id_rs2));

        // When hazard detected:
        //   pc_write    = 0  -> PC register holds its current value (re-fetch same instr)
        //   if_id_write = 0  -> IF/ID holds its current value (re-decode same instr)
        //   id_ex_flush = 1  -> ID/EX is cleared to NOP (bubble injected)
        pc_write    = ~load_use_hazard;
        if_id_write = ~load_use_hazard;
        id_ex_flush =  load_use_hazard;
    end

endmodule
```

**ID/EX flush logic in the pipeline register update:**
```systemverilog
always_ff @(posedge clk) begin
    if (id_ex_flush) begin
        // Zero all control signals -> NOP bubble
        id_ex_reg_write  <= 1'b0;
        id_ex_mem_read   <= 1'b0;
        id_ex_mem_write  <= 1'b0;
        id_ex_branch     <= 1'b0;
        id_ex_alu_src    <= 1'b0;
        id_ex_alu_op     <= 2'b00;
        id_ex_mem_to_reg <= 1'b0;
        // Data fields: technically don't matter (control signals are zeroed),
        // but zeroing them prevents X-propagation in simulation
        id_ex_rs1_data   <= 32'b0;
        id_ex_rs2_data   <= 32'b0;
        id_ex_imm        <= 32'b0;
        id_ex_rd         <= 5'b0;
    end else begin
        // Normal update from ID stage outputs
        id_ex_reg_write  <= ctrl_reg_write;
        id_ex_mem_read   <= ctrl_mem_read;
        // ... (all other fields)
    end
end
```

**IF/ID freeze logic:**
```systemverilog
always_ff @(posedge clk) begin
    if (if_id_write) begin
        // Normal update: latch newly fetched instruction
        if_id_instruction <= imem_read_data;
        if_id_pc_plus4    <= pc_plus4;
    end
    // else: if_id_write == 0 -> register holds its current value (implicit freeze)
end
```

**PC freeze logic:**
```systemverilog
always_ff @(posedge clk) begin
    if (pc_write) begin
        pc <= next_pc;   // next_pc from branch/jump/sequential logic
    end
    // else: pc holds its current value
end
```

**Interaction with forwarding:** After the stall cycle, the pipeline resumes. The LW instruction is now in the MEM stage (EX/MEM register). The hazard detection unit no longer asserts a stall (ID/EX.MemRead is now 0 for the bubble that was inserted). The forwarding unit then detects that EX/MEM.rd matches the consumer's rs1 or rs2, and asserts the EX/MEM forward path. No additional stall is needed.

---

### Question A3
**A RISC-V processor is observed to have higher CPI than expected. Profiling reveals 18% of instruction pairs are load-use pairs and the compiler was built without `-O2`. Explain three compiler-level and two microarchitecture-level strategies to reduce the load-use penalty impact. Quantify the CPI improvement for each where possible.**

**Answer:**

**Baseline CPI contribution from load-use:**
```
CPI_load_use = 0.18 * 1 cycle = 0.18 cycles/instruction
```

**Compiler strategies:**

**1. Load scheduling (instruction reordering):**
The compiler reorders independent instructions to fill the load-delay slot. For example:

```
Before scheduling:         After scheduling:
  LW  x1, 0(x2)             LW  x1, 0(x2)
  ADD x3, x1, x4            ADD x5, x6, x7   <- independent instruction moved here
  ADD x5, x6, x7            ADD x3, x1, x4   <- load consumer now 1 instr later
```

If a suitable independent instruction exists for 70% of load-use pairs, the effective hazard rate drops from 18% to 5.4%:
```
CPI_load_use after scheduling = 0.054 * 1 = 0.054 cycles/instruction
CPI improvement = 0.18 - 0.054 = 0.126 cycles/instruction  (~50% reduction for this component)
```
This is the primary benefit of `-O2 -fschedule-insns2` in GCC.

**2. Software pipelining / loop unrolling:**
For tight loops dominated by loads (e.g., array traversal), the compiler unrolls the loop and interleaves load-use pairs from different loop iterations, eliminating nearly all stalls:

```
Unrolled by 2:
  LW  x1, 0(base)
  LW  x2, 4(base)     <- start loading next element while x1 is still in-flight
  ADD x3, x1, x4      <- x1 now available (1 instruction gap)
  ADD x5, x2, x4
```

For perfectly pipelined loops, load-use stalls can approach zero.

**3. Use of indexed addressing and instruction combination:**
Some load patterns can be replaced by load-and-compute instructions if the target supports custom extensions, or by reorganising data layout to enable SIMD/vector loads that naturally have wider spacing between load and use.

**Microarchitecture strategies:**

**4. Non-blocking cache with out-of-order load execution:**
Even in an otherwise in-order core, a non-blocking L1 data cache can allow a load to proceed speculatively and resolve its value when the cache responds, without forcing a full pipeline stall. The load result is written directly to a load buffer from which forwarding occurs. This reduces the effective load-use penalty from 1 cycle (for a cache hit) to potentially 0 cycles if the result returns before the consumer reaches EX.

CPI impact: Eliminates up to 100% of the load-use stalls that hit in the L1 cache (cache hit rate typically 95%+), giving:
```
CPI_load_use reduced = 0.18 * (1 - 0.95) * 1 = 0.009  (only L1 misses still stall)
Improvement = 0.18 - 0.009 = 0.171 cycles/instruction
```

**5. Load-to-use bypass with 2-stage decode (early load completion):**
Some designs split the MEM stage into tag-lookup and data-read sub-stages. If the data cache returns data early enough (before the end of what was the single MEM stage), forwarding can be timed to meet the EX stage of the consumer without a stall. This requires careful timing closure and a modified forwarding path from the early-data output of the cache to the ALU. Effective when the cache SRAM access time is short enough to complete within 60-70% of the clock period.
