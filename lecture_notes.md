# Week 2 - Lecture 2

## Classifying ISAs

Complexity:

- CISC: complex instruction set computing
- RISC: relaxed instruction set computing

### CISC

- small register file (e.g. 8 registers)
- large number of instructions
- variable-length instructions (optimize compiled program size?)
- complex instructions that execute many simpler instructions (e.g. `rep`)

Design motivations:

- small register file: expensive transistors
- reducing program size: expensive memory
- complex instructions: simplify assembly programming (compilers sucked, ppl programmed directly in asm)

### RISC

- smaller instruction set (hence larger compiler programs)
- larger register file (e.g. 32 registers)
- fixed-length instruction
- simple addressing mode:

Design updates motivated by cheaper memory/hardware.

- simpler is faster: many simpler instructions faster than one complex one
- memory got cheaper
- transistors cheaper
- compilers got better (complexity was moved to the compiler)

Intel dilemma:

- cannot drop CISC for legacy reasons
- implement translation from CISC to RISC

Google TPU Matrix ISA similar to CISC:

- complex instructions for matmul etc., execute over 64+ cycles

### VLIW

Very Long Instruction Word

- encodes multiple instructions in a single word ("bundle")
- execute each operation on a different execution unit
- compiler must ensure no dependencies between instructions in same word, for parallelism

Decoding complex words:

- need program counter as well as instruction index to fetch instructions

Motivation:

- simplify hardware: move complexity of scheduling instructions to the compiler
    - contrast with OOO execution, hardware-bound (?)
    - increase flexibility of scheduling by moving it to software (compiler?)
- increased parallelism
    - instruction-level parallelism

> [constructing a bundle]
>
> [add mul load]
> ALU fixed-latency pipeline
> Move multiplier outside of ALU:
> send the add to the ALU, mul to the multiplier
>
> - don't bottleneck ALU with costly mult
> - some multipliers are variable-latency
> - leverage parallelism

Shortcomings:

- compiler becomes very complex
- portability of ISA
    - too coupled with hardware (e.g. instruction latency)
    - defeats purpose of ISA
    - information about hardware (cycle count of different operations) need to be visible to software
- turns out, the parallelizable bundles are scarce in real code
    - if bundle has two adds, not parallelizable: both adds must be sent to the ALU
    - data dependencies are common
    - noops and the like result in wasted fetches of large words

### Vector ISAs

SIMD: single instruction multiple data
`vadd v3 <- v1 + v2`
one instruction, each operand contains several data

- leverages parallelism (vec operations are parallel)
    - fetch multiple data at the same time
- single instruction: smaller program, fit more instruction into the instruction cache

"replicate ALU for each element of the vec add"

Shortcomings:

- memory bandwidth
    - increased bandwidth requirements
- control flow limitations
    - divergent conditional values
    - jumping might be conditional on values of vector element
- programmability
    - writing vectorized programs is hard!
        - vectorization not always possible
        - compilers not able to automatically vectorize
    - data alignment restrictions
        - how is data loaded into a vector?
        - if a single load is performed, then data must have 16 byte alignment
        - if eight smaller loads are performed, then data must have 4 byte alignment (better, more flexible)

## Instructions

Instruction encodings follow fixed patterns -> allows parallelized decoding

- if (e.g.) `add` and `sub` were structured differently, decoding the (e.g.) 'read from' part would depend on which of `add` and `sub` we received
- if the rd is always in the same place, then the parallelized rd unit in the decoder can be simpler

PARALLELIZATION NEEDS MODULARITY

Can read instruction type from prefix of opcode (don't need to know the whole op to know its type)

# Week 3 - lecture 2

Basic pipeline loop:

```
fetch -> decode -> evaluate -> update
```

## CSI: Control and Status Instructions

CSIs are ways for software to read/write machine status.

Timers and counters

- 64 bits
    - latency in cpu pipeline means timing is subtle: clock ticks is stored as 64 bits, so needs to fetches on 32 bit machine, but clock might move between fetches.
    - cache misses might increase fetch time
    - solution: loop: fetch high, fetch low, fetch high again; loop if high has changed (overflow)
        - note: time fetches will not be perfectly precise (depends on loop iterations, clock cycles for executing loop operations, etc.)

- privilege levels
    - define levels of access that software has to processor and system
        - enables access control and security (isolation)
    - levels:
        - User-level (U) 0
        - Hypervisor (H) 1
        - Supervisor (S) 2
        - Machine-level (M) 3

e.g.

```
|   user   |    |    user   | <- user-level
| linux vm |    | windows vm| <- supervisor
|           mac os          | <- hypervisor
|          machine          | <- machine level
```

CSR instructions encode privilege level -> no need to execute instruction to know privilege level

- contrast with storing a table encoding instruction privilege separately
    - would require hardware storage of table
    - not modular: any privilege change to the ISA would require rewriting the table, instead of just encoding the new privilege

## Registers

- `x0` register hardcoded to `0`
    - `0` is commonly used value
    - implements `NOP`: `add x0, x0, x0`
    - controlling writeback for `jal` (if `rd != x0`, increment `pc`)
- `pc` is a hidden register
    - accessing `pc` through read/store?
        - `auipc rd, 0`
        - `jal rd, 4`

ABI: application binary interface
Calling convention:

- arguments: a0-7
- return: a0
- temporaries: t0-6
- saved registers: s0-11

Pseudo instructions:

- assembly-level instructions that do not correspond to machine-level instructions
    - syntactic sugar for assembled instructions
- `li s1 100` is a pseudo instruction: corresponds to `addi s1, 100`
    - note that if the immediate operand is too large to fit in the `addi` immediate field, `li` would be unrolled into two instructions.
- pseudo instructions are "unrolled" during assembly; this may break instruction pointer arithmetic

### Stack

Values are pushed onto the stack when they need to be saved or when there are more arguments than available registers.

### Global pointer

Tracks global variables in program (e.g. `static` variables in C)

### Thread pointer

Used to efficiently access thread-local storage (TLS)

e.g. each thread manages a `thread_local` counter; the thread pointer refers to the base address of the counters

> note: `HARTID == THREADID`

# Week 4 - Lecture 1

## Circuit design

### Combinational circuits

Memoryless: output does not depend on past inputs/state/outputs, but only on current input

- multiplexer:
    - essentially a switch

### Sequential circuits

- maintain state; output depends on current input and prior state
- depends on clock
- e.g. D flip flop:
    - implements registers
    - resets propagation delay

### Propagation delay and critical path

- propagation delay (Tp): time needed for gate to respond to change in inputs
- critical path (Tmax): longest propagation delay in the circuit between I/O or registers
- matters to know whether we can "drive" the circuit using the clock
    - clock frequency should be f = 1/Tmax

### FSMs for circuits

Mealy type FSM:

- (state, input) -> output
- pros:
    - input contributes directly to output
- con:
    - inherits delay from input

Moore type FSM:

- (state) -> output
- pros:
    - fast: input is passed through flip flop, hence resets propagation delay
- cons:
    - lots of states needed

## CPU Pipelineing

## Datapath

All state components take clock input.

1. program counter: stores address of next instruction
2. instruction memory
3. register file
4. memory

### pc

- one component increments pc by 4 every clock cycle (to fetch next instruction) (in parallel)
- another fetches instruction from memory

### register file

- takes register IDs and outputs the content of these registers
- two read ports: some instructions include two source registers

- decoding register address: no logic needed because each field gets sent to the same place regardless of the instruction

### branch operations

- note the 1 bit left shift: we know memory addresses are 4byte aligned, so can decode immediate into half the offset
    - why not shift left by 2?
    - extensibility, compatibility with compressed ISA

### memory operations

- note that

## Control path

- control module
    - converts optype to an array of signals controlling the datapath
- alu control module

## Single cycle CPU

- all cpu operations are executed in one cycle
    - very simple
- huge propagation delay

## multi-cycle CPU

- insert registers between main components
