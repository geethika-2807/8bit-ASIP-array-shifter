# 8-Bit ASIP CPU — Circular Array Shifter

**A custom processor built from scratch that transforms a C program into working hardware**

---

## What We Built

We designed and implemented a complete 8-bit microprocessor in Logisim Evolution that performs a continuous circular left shift on a 10-element array. This isn't a simulation of a pre-existing CPU — it's a custom processor built from logic gates, registers, and memory, executing a real algorithm you could write in C.

The CPU runs this operation endlessly:
- Takes an array: `[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]`
- Shifts everything left by one position
- Wraps the first element to the end: `[1, 2, 3, 4, 5, 6, 7, 8, 9, 0]`
- Repeats forever

Every loop, array access, and comparison in the C code has a hardware equivalent in our CPU.

---

## Why We Built This

Modern processors are incredibly complex — billions of transistors, deep pipelines, sophisticated branch predictors. But at their core, they all do the same thing: fetch instructions, decode them, execute operations, and write back results.

This project strips away all that complexity to reveal the fundamentals. By building a CPU that does one specific task (circular array shifting), we understand:

- How software becomes hardware signals
- How instructions are fetched, decoded, and executed
- How control logic coordinates datapath components
- How memory, registers, and ALU work together

It's hands-on computer architecture — the best way to understand what's happening inside the device you're reading this on.

---

## How We Did It

**The Journey from C to Hardware:**

1. Started with a simple C program that shifts an array
2. Broke down each line into basic operations (load, store, add, compare, jump)
3. Designed a custom Instruction Set Architecture (ISA) with 8 instructions
4. Built the datapath: registers, ALU, memory, and control logic
5. Wrote the machine code program
6. Connected everything in Logisim Evolution
7. Tested and debugged until it ran correctly

The result is a Harvard-architecture CPU with 4 registers, microcode control, and single-cycle execution — all built from fundamental logic components.

---

## The Challenge

Our CPU executes this algorithm continuously:

```c
int a[10] = {0,1,2,3,4,5,6,7,8,9};
int i = 0, j = 0;

void main() {
    while(1) {
        j = a[0];           // Save first element
        for(i = 0; i < 9; i++)
            a[i] = a[i+1];  // Shift everything left
        a[9] = j;           // Wrap first element to end
    }
}
```

**What "circular left shift" means:**

```
Start:  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
         ↓
Step 1: [1, 2, 3, 4, 5, 6, 7, 8, 9, 0]  (0 wrapped to end)
         ↓
Step 2: [2, 3, 4, 5, 6, 7, 8, 9, 0, 1]  (1 wrapped to end)
         ↓
...and so on, infinitely
```

---

## Inside the CPU

Our processor uses a **Harvard architecture** — separate memory for instructions and data. This allows the CPU to fetch the next instruction while executing the current one, improving efficiency.

### The Components

**1. Program Counter (PC)**
The PC is an 8-bit register that holds the address of the current instruction. It automatically increments to point to the next instruction, or jumps to a new address when branch instructions execute. Think of it as the CPU's "bookmark" in the program.

**2. Instruction Register (IR)**

A 16-bit register positioned between Program ROM and the instruction decoder. It latches the current instruction on every rising clock edge, holding it stable for the entire execution cycle.

**Why it matters:** Without the IR, the instruction would change mid-cycle when the Program Counter updates to the next address. The IR ensures the instruction remains constant from fetch through write-back, making this a proper single-cycle CPU design.

**How it works:**
- Input: 16-bit instruction from Program ROM
- Clock: Connected to the system clock (rising edge triggered)
- Output: Stable 16-bit instruction fed to the splitter/decoder
- The splitter breaks this into opcode (bits 15-12), register address (bits 9-8), and immediate (bits 7-0)

The IR is essential for correct operation — it's the difference between a working CPU and one that executes garbage instructions.

**3. Control Unit**
The "brain" of the CPU. It's a 16-entry microcode ROM that takes the 4-bit opcode and outputs an 8-bit control word. Each bit in this word activates a specific part of the datapath:
- Should we write to a register?
- Should we read from memory?
- Should the ALU add or subtract?
- Should we jump?

This approach is cleaner than complex logic gates — to add a new instruction, we just add a new entry to the ROM.

**4. Register File**
Four 8-bit general-purpose registers (R0, R1, R2, R3), each with a specific job:
- **R0**: Temporary storage during shift operations
- **R1**: Preserves `a[0]` during the shift loop (for wrap-around)
- **R2**: Loop counter (counts from 0 to 9)
- **R3**: Dedicated memory pointer (always holds the current RAM address)

R3 is hardwired as the memory address source, so we don't need a separate address register.

**5. ALU (Arithmetic Logic Unit)**
The computational engine that performs:
- **ADD**: Incrementing pointers and the loop counter
- **SUB**: Adjusting pointers and performing comparisons
- **CMP**: Comparing values (subtracts without writing, just sets a flag)

The ALU uses a multiplexer to select between adder and subtractor outputs. An 8-input NOR gate monitors all output bits to generate the zero flag (ZF=1 when result is zero).

**6. Memory System**

Our CPU uses two separate memory components following the Harvard architecture:

**Program ROM (Read-Only Memory)**
- 256 x 16-bit storage holding our 13-instruction program
- Addressed by the Program Counter (PC)
- Returns the 16-bit instruction for the current address
- Contents are fixed at design time (not modified during execution)
- Organized as: 8-bit address → 16-bit instruction word

The Program ROM outputs the instruction corresponding to the PC address. This instruction is then latched into the Instruction Register for decoding and execution.

**Data RAM (Random Access Memory)**
- 256 x 8-bit storage holding the 10-element array
- Addressed by the R3 register (our dedicated memory pointer)
- Supports both read and write operations during execution
- Initialized with test values before simulation starts
- Organized as: 8-bit address (from R3) → 8-bit data byte

The Data RAM stores our array a[0] through a[9]. During execution, the CPU reads values for shifting and writes shifted values back to memory. The R3 register always holds the current address being accessed.

**7. Write-Back MUX**

A 4-to-1 multiplexer that determines what data gets written back to the register file. After an instruction executes, the result must be routed back to a register. The Write-Back MUX selects the source:

- **Input 0 (ALU Result)**: Used by ADD, SUB, and CMP instructions. The ALU performs the operation and the result goes back to the register.
- **Input 1 (Immediate Value)**: Used by LDI instructions. The 8-bit immediate value from the instruction is written directly to the register.
- **Input 2 (RAM Data)**: Used by LD instructions. Data read from RAM is written to the register.
- **Input 3 (Zero)**: Unused in our program, but tied to constant 0 for completeness.

The selection is controlled by two bits from the Control Unit: DataReg and ImmReg. These bits form a 2-bit select signal that chooses which input passes through to the register file's Data_In pin.

---

## The Instruction Set

Our CPU uses 16-bit instructions with this format:

```
[Opcode: 4 bits][Register: 2 bits][Unused: 2 bits][Immediate: 8 bits]
```

| Opcode | Instruction | What It Does | Why We Need It |
|--------|-------------|--------------|----------------|
| 0000 | LDI | Load immediate value into register | Initialize pointers and counters |
| 0001 | LD | Load from RAM (at R3 address) | Read array elements |
| 0010 | ST | Store to RAM (at R3 address) | Write shifted values |
| 0011 | ADD | Add immediate to register | Increment pointers and counter |
| 0100 | SUB | Subtract immediate from register | Adjust pointers |
| 0101 | CMP | Compare (subtract, set flag only) | Check loop condition |
| 0110 | JNE | Jump if not equal (ZF = 0) | Repeat the shift loop |
| 0111 | JMP | Unconditional jump | Restart the program |

Only 8 instructions — that's all we need for this task.

---

## The Program

Here's the 13-instruction machine code that makes the array shift:

| Address | Assembly | Hex Code | What's Happening |
|---------|----------|----------|------------------|
| 0x00 | LDI R3, 0 | 0xC00 | Set memory pointer to start of array |
| 0x01 | LD R1, 0 | 0x1400 | Save a[0] in R1 (we'll need it later) |
| 0x02 | LDI R2, 0 | 0x0800 | Reset loop counter to 0 |
| 0x03 | ADD R3, 1 | 0x3C01 | Move pointer to a[i+1] |
| 0x04 | LD R0, 0 | 0x1000 | Load a[i+1] into temporary R0 |
| 0x05 | SUB R3, 1 | 0x4C01 | Move pointer back to a[i] |
| 0x06 | ST R0, 0 | 0x2000 | Store R0 into a[i] (the shift!) |
| 0x07 | ADD R2, 1 | 0x3801 | Increment loop counter |
| 0x08 | ADD R3, 1 | 0x3C01 | Advance pointer for next iteration |
| 0x09 | CMP R2, 9 | 0x5809 | Have we shifted 9 elements yet? |
| 0x0A | JNE 0x03 | 0x6003 | If not, repeat the shift loop |
| 0x0B | ST R1, 0 | 0x2400 | Write saved a[0] into a[9] (wrap-around) |
| 0x0C | JMP 0x00 | 0x7000 | Jump to start (infinite loop) |

---

## How It Works: Step-by-Step

### Phase 1: Initialization (3 instructions)

The CPU sets up its working state:
1. **LDI R3, 0**: Point to the start of the array
2. **LD R1, 0**: Save a[0] (we'll wrap this to the end later)
3. **LDI R2, 0**: Reset the loop counter

### Phase 2: The Shift Loop (9 iterations)

Each iteration shifts one element left:

**Iteration example (shifting a[0] to a[8]):**
1. **ADD R3, 1**: Point to a[1]
2. **LD R0, 0**: Load a[1] into R0
3. **SUB R3, 1**: Point back to a[0]
4. **ST R0, 0**: Write a[1] into a[0] ← **First shift complete**
5. **ADD R2, 1**: Counter = 1
6. **ADD R3, 1**: Point to a[2] for next iteration
7. **CMP R2, 9**: Check if we're done
8. **JNE 0x03**: If counter < 9, repeat

This repeats 9 times, shifting a[1]→a[0], a[2]→a[1], ..., a[9]→a[8].

### Phase 3: Wrap-Around (2 instructions)

After the loop:
1. **ST R1, 0**: Write the saved a[0] into a[9]
2. **JMP 0x00**: Jump back to start

**Result:** The array has shifted left by one position!

```
Before: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
After:  [1, 2, 3, 4, 5, 6, 7, 8, 9, 0]
```

The CPU then repeats this forever, continuously rotating the array.

---

## Running the Simulation

### Prerequisites
- **Logisim Evolution** v4.1.0 or later ([Download here](https://github.com/logisim-evolution/logisim-evolution))

### Setup Steps

1. **Open the project**: Load `cpu.circ` in Logisim Evolution
2. **Initialize the array**: Right-click Data RAM → Edit Contents → Enter values (e.g., `00 01 02 03 04 05 06 07 08 09`)
3. **Set simulation speed**: Simulate → Simulation Frequency → 8 Hz (slow enough to watch)
4. **Start the clock**: Simulate → Auto-Tick Enabled (or press Ctrl+K)
5. **Watch it work**: The array shifts in RAM after every 13 clock cycles

### For Demonstrations

To show the CPU with custom arrays:
1. Stop simulation and reset (Ctrl+R)
2. Load your custom values into Data RAM
3. Set PC to 0
4. Restart the clock
5. Use probes to show R3 (memory pointer) and RAM values

**Expected behavior:** After one full cycle (13 ticks), the array shifts left by one. After 10 cycles, it returns to the original state.

---

## Design Decisions

We made several key architectural choices that shaped the CPU:

### Instruction Register (IR)

**What it is:** A 16-bit register between Program ROM and the instruction decoder.

**Why we need it:** Without the IR, the instruction would change mid-cycle when the Program Counter updates. The IR latches the instruction on every clock edge, holding it stable throughout execution. This makes our design architecturally correct for a single-cycle CPU.

### Microcode Control Unit

**What it is:** A 16-entry ROM that converts 4-bit opcodes into 8-bit control words.

**Why we chose it:** Instead of complex combinational logic generating control signals, we store them in ROM. This makes the design:
- **Easier to modify**: Add new instructions by adding ROM entries
- **Easier to debug**: Inspect ROM contents to see control signals
- **Cleaner**: No tangled logic gates

### Dedicated R3 Memory Pointer

**What it is:** R3 is hardwired as the source for RAM addresses.

**Why we chose it:** Eliminates the need for a separate address register. All memory operations automatically use R3, so pointer arithmetic is just incrementing/decrementing R3. This simplifies the datapath and leaves R0, R1, R2 for computation.

### Combinational Branch Logic

**What it is:** The branch path from zero flag → AND gate → PC MUX has no D flip-flop.

**Why we chose it:** An early design had a DFF that caused a 1-cycle delay on conditional jumps. The JNE instruction would jump one instruction late, breaking the loop. Removing the DFF makes the branch combinational, so it takes effect immediately.

---

## Screenshots

*Add screenshots of your Logisim design here to showcase the implementation*

### Main Datapath
![Main Datapath](screenshots/main-datapath.png)
*The complete CPU showing all components: Program ROM, IR, Control Unit, Register File, ALU, Data RAM, and Write-Back MUX connected together*

### Register File Sub-circuit
![Register File](screenshots/register-file.png)
*The 4 registers (R0-R3) with write decoder, AND gates for write enable, and read MUX*

### ALU Sub-circuit
![ALU](screenshots/alu.png)
*Adder, subtractor, output MUX, and zero flag circuit (8-input NOR gate)*

### Control Unit Sub-circuit
![Control Unit](screenshots/control-unit.png)
*Microcode ROM with opcode input and 8 control signal outputs*


---

## Technical Specifications

| Feature | Specification |
|---------|---------------|
| **Architecture** | Harvard (separate instruction/data memory) |
| **Data Width** | 8 bits |
| **Instruction Width** | 16 bits |
| **Registers** | 4 x 8-bit general-purpose (R0-R3) |
| **ISA Size** | 8 instructions |
| **Address Space** | 256 bytes (8-bit address bus) |
| **Program Memory** | 256 x 16-bit ROM |
| **Data Memory** | 256 x 8-bit RAM |
| **Control** | 8-bit microcode word from 16-entry ROM |
| **ALU Operations** | ADD, SUB, CMP |
| **Execution Model** | Single-cycle |
| **Clock Cycles per Shift** | 13 |
| **Total Instructions** | 13 |

---

## What We Learned

This project taught us that processors aren't magic — they're just incredibly organized logic. Every C construct has a hardware equivalent:

- **Loops** → Branch logic + counter registers
- **Array access** → Memory operations + pointer arithmetic
- **Conditionals** → Comparisons + conditional jumps
- **Variables** → Register transfers + ALU operations

Building this CPU from first principles gave us a deep understanding of:
- How the fetch-decode-execute cycle works in hardware
- Why architecture choices (Harvard vs. von Neumann) matter
- How microcode simplifies control logic
- The relationship between software and hardware

**The key insight:** A CPU is just a very disciplined machine that moves data between registers and memory based on simple rules. Complexity emerges from repeating these simple operations billions of times per second.

---

## Future Enhancements

This CPU is intentionally minimal — it only does what it needs to. But there's room to grow:

- **Expand the ISA**: Add MUL, DIV, logical operations (AND, OR, XOR)
- **Status register**: Track carry, overflow, and negative flags
- **Pipelining**: Execute multiple instructions simultaneously for better performance
- **Interrupts**: Handle external events
- **Assembler**: Automatically convert assembly code to machine code
- **Larger arrays**: Expand address bus beyond 8 bits
- **I/O peripherals**: Add display, keyboard, or other interfaces

---

## Conclusion

This project proves that you don't need billions of transistors to build a working CPU. With just logic gates, registers, and memory, we created a processor that executes a complete algorithm — continuously, correctly, and reliably.

Understanding this design changes how you think about computers. Every app, every website, every program you run is ultimately just instructions moving data through circuits, just like our CPU does with those 13 instructions.

**If you understood how this CPU works, you understand how all computers work.**

---

*Built with Logisim Evolution | Harvard Architecture | 8-Bit Datapath | Custom ISA | Single-Cycle Execution*