# Chapter 1 - Abstractions

## Time

Response time:

- time it takes to complete a given task.
- performance is 1/execution_time

Different definitions of time:

- wall clock time/response time/elapsed time: real world time, takes into account everything the computer does to complete the task.
- CPU time: time spent by CPU on that particular time (excludes I/O, paused execution, etc.)

**Clock cycle**: basic unit of time at the processor level. E.g. 250 picoseconds. Inverse of **clock rate**.

CPI - clock cycles per instruction: how many clock cycles to execute a given instruction.

Some relationships:

- CPU execution time for X = CPU clock cycles for X x clock cycle for CPU
- CPU clock cycles for X = instructions count for X x average CPI
- CPU time = instruction count x CPI x clock cycle time

Intuitively,
time/program = instructions/program x clock cycles/instruction x time/clock cycle

Thus key factors for the performance of a program are

- instruction count (program dependent)
- CPI (semi-dependent?)
- clock cycle (independent)

## Energy

Energy mostly consumed in switching the state of transistors (**dynamic energy**).

- energy of a pulse 0 -> 1 -> 0 (or 1 -> 0 -> 1): energy ~ capacitive load x voltage^2
- energy of a transition (e.g. 0 -> 1): energy ~ 0.5 x capacitive load x voltage^2
- power per transistor: transition energy x transition frequency

Transition frequency is a function of clock rate

Power wall:

- power requires cooling
- increasing clock rate increases transition frequency and thus power
- to keep power low while increasing clock rate, must reduce voltage or capacitive load
- but voltage is hard to reduce below a given point (resulting in "leaky transistors")

## Multiprocessors

Instead of increasing the performance (clock rate) of single processors, designers switched to increasing the number of processors.

Key challenges of parallelism:

- scheduling
- load-balancing
- synchronization
- communication

## Amdhal's law

new_time = old_time_affected / improvement + old_time_unaffected

# Chapter 2 - Instructions

Numeric versions of instructions called **machine language** (compare with **assembly language**-a higher-level notation for the instructions)

- each ISA (spec for machine language) has a corresponding assembly language (human-readable translation)

**Instruction format**: layout of the components of an instruction

- RISC-V: all instructions have the same length
- different instructions therefore require different formats
    - in particular, a load instruction requires a base address (stored in register), an offset (immediate value), and a destination (register), instead of two register operands and a destination as in an ADD operation.
    - this gives a greater range for the offset than possible in a register field, since the latter are only 5 bits long

## Memory convention

By convention

- the stack pointer is initialized to 000 003f ffff fff0 and grows downward.
- the program text starts at 0000 0000 0040 0000 and is filled upward.
- the static data starts immediately after the text.
- dynamic data starts after static data and constitutes the heap.

## Addressing modes

- Immediate addressing
    - operand is a constant encoded in the instruction
- Register addressing
    - operand is stored in a register; register in encoded in instruction
- Base/displacement addressing
    - operand is stored in memory; address of operand is computed as immediate + register
    - i.e.: address is [register] + immediate offset
- PC-relative addressing
    - operand is stored in memory; address of operand is computed as program counter + immediate

## Synchronizing parallelized instructions

## Stages of compilation

- compilation
    - translates from source code to assembly code.
- assembly
    - translates assembly code to machine code.
    - assembly code can have instructions that do not correspond to machine code
        - e.g. li x9, 123 loads immediate value 123 in register 9, but is implemented as addi x9, x0, 123
        - also, can express numbers in several bases
    - produces an **object file**
        - object file header: size and position of other parts of object file
        - text segment: machine language code
        - static data
        - relocation information: identifies which instructions and data words depend on absolute addresses
        - symbol table: remaining undefined labels (e.g. external references)
        - debugging information
- linking
    - links together separate object files (e.g. library routines), allowing for modular compilation
    - three steps:
        - places code and data modules symbolically in memory
        - determines the addresses of data and instruction labels
        - patches internal and external references
    - produces an **executable file**
        - executable has no unresolved references
- loading
    - reads executable file header to determine size of the text and data segments
    - creates an address space large enough for the text and data
    - copies the instructions and data from the executable file into memory
    - copies the parameters (if any) to the main program onto the stack
    - initializes the processor registers and sets the stack pointer to the first free location
    - branches to a start-up routine that copies the parameters into the argument registers and calls the main routine of the program.
        - when the main routine returns, the start-up routine terminates the program with an `exit` system call.

### Static linking

# Chapter 2

## Offsets

Branch instructions (e.g. bne) and function call instructions (e.g. jal) specify the jump address using an immediate value (jalr uses a value stored in a register).

- Branch instructions have space for a 12-bit immediate (since they must also specify two registers to compare).
- jal has space for a 20-bit immediate (since it only needs to specify a single register to store the return address, and doesn't need a funct code to specify the type of branching)

- If the jump address were identical to the immediate, programs would be limited in size (at most 2^20 addresses).
- instead, the jump address is specified as offset + base. The offset is the instruction immediate. The base is the program counter.

- for very long jumps (that are more than 2^20 away from the program counter), we can build a 32-bit offset by using `lui rx upper_bits` followed by `jalr x1 lower_bits(rx)`.

- instructions are 32 bits (4 bytes) long, whereas memory addresses are 1 byte. Hence, could have increased range by encoding distance in instructions rather than words.
    - but for compatibility with 2 byte instructions, distance is encoded in half instructions (i.e. units of 2 bytes)

- long distance branches are dealt with by flipping the branch condition (e.g. `bne` -> `beq`), which then decides whether to skip a long-range `jal`.

# Chapter 4 - The Processor

## Hazards

Hazards occur when the next instruction cannot execute in the next clock cycle.

- structural hazard:
    - hardware cannot support a combination of instructions, preventing pipelining
    - easyish to prevent in pipeline-designed processors
- data hazard:
    - dependence of an earlier stage of an instruction on some later stage of prior instruction
        - second instruction must wait until first instruction has finished the relevant stage
    - can be resolved by forwarding: don't wait for i1 to reach the write stage, just forward output of ALU directly to the register read stage of i2.
        - even with forwarding, might need to stall: some dependencies are more than one stage long
        - `add` needs at least one instruction between a `lw` on which it depends for forwarding to work.
            - because `add` depends on `lw` to write to registers for its input to the ALU; forwarding can happen once `lw` has read from memory; memory access is one stage after the ALU; to ensure that the memory access is done before the ALU starts, need the memory access to happen two cycles before the ALU execute. By default, `lw` will start one cycle ahead; placing an extra instruction in between ensures forwarding can take place.
        - if `sw` depends on `add`, forwarding can take place straightaway, since `sw` depends at stage MEM on the output of the EX stage of `add`, which is at minimum two cycles behind
- control hazard:
    - a branch decision depends on the results of a prior instruction.
        - can either stall - slow
        - or predict whether the branch will be taken
            - fast and cool
