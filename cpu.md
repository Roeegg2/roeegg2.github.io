# CPU

**_Written by Roee Toledano_**

## Introduction

The NES has a slightly modified MOS 6502 CPU.
This CPU was quite popular back in the day, so there is plenty of detailed documentation out there, but 
I will provide here a basic explanation of how it works.

## Terminology

There isn't really any fancy emulation/NES terminology used here, just computer stuff. I will explain some terms very simply, if you want a more in depth explanation you can look these things up online.

- Registers - small, fast areas of storage the CPU can fetch data from and write data to.

- RAM (Random Access Memory) - a type of memory the CPU can read data to and write data from. On most modern computers the code for a game/program will be stored on a txt section here, but on the NES this is stored on PRG ROM (I'll explain on a future writeup about mappers and cartridges)

- MMIO (Memory mapped I/O) - Roughly speaking, this is mapping a certain memory address to a device, so by reading/writing that address we can transfer data from/to that device. Look it up online for a more detailed explanation.

- ISR (short for interrupt service routine) - the function called to handle an interrupt. Different interrupts have different ISRs.

- Interrupt Vector - An interrupt vector is a memory address which contains the address of the ISR to be called for this specific interrupt.

## Part 1 - What does a CPU do?

At the end of the day, CPU is just a component which receives some data, does some calculations, and outputs some data.
The CPU is composed of ALU, Registers, and a clock

Roughly, each cycle the CPU does one of these operations:

1. Fetch - fetch an instruction
2. Decode - figure out what that instruction does
3. Execute - do that instruction

The binary code the CPU needs to execute looks something like this:

```psuedo
opcode operand1 operand2 opcode opcode operand1 ... etc
```

An opcode is a number unique to each operation the CPU perform, so the CPU knows what it should should do, and how to get the data used by that instruction. Similar instructions could have the different opcodes, because they uses different addressing modes.

**Operands**, are data that the operation needs as parameters. Somewhat similar to function arguments in high level languages such as C, Java, Python, etc.
Each operation's operand number range from 0 (the minimum amount) to 2 (the maximum amount)

An a addressing mode, dictates a certain way the CPU should get and treat the data in the operands.

Let's take for example the following code snippet:

```assembly
lda #$11
sta $200
```

All that code does is load the value from `$11` into A register, then store the value in A in address `$200`.

The assembled code will be:

```psuedo
a9 11 02 8d 00 02
```

the `a9` here is the opcode for the `lda (load a)` instruction, using the _immediate_ addressing mode.
the `11` is `lda`s first operand
the `8d` is the opcode for the `sta` instruction, using _absolute_ addressing modes, and both
the `00` is `sta`s first operand
the `02` is `sta`s second operand.

(The reason the data 200 is stored reversed, as 00 02 is because the 6502 is little endian. Look up online what this means)

As I said before, an addressing mode is the way the CPU interprets the operands of the instruction.
As we have seen above, operands on different operations have different meaning.
In our example, the `lda` instruction used `11` as a hex-form number that should be loaded to A as is. Without addressing modes, the CPU could have also interpreted `11` as a decimal-form number, that should be used as an address from which data should fetched and stored in A.
These 2 interpretations have very different meanings.
So how does the CPU knows how it should use that data? The answer is addressing modes.

Each addressing mode of the CPU is a different way the CPU should interpret the values of the operands. I won't list and explain them all here, but as we can infer from the example above, _immediate_ addressing means "using the raw value as is" and _absolute_ means "use the 2 operands to make up an address" (in our case the instruction was `sta`, so that means store the contents of A in the address `$0200`)

You can check out all the 6502 instructions and the available addressing modes [here](https://www.masswerk.at/6502/6502_instruction_set.html)

## The ALU

ALU (Arithmetic Logic Unit) - that is the part of the CPU that actually performs the calculations. It takes in data some data, and outputs it out to a register called the accumulator.

<!-- FINISH WRITING HERE -->

## Registers

The 6502 has a few registers:

- X - a 1 byte wide utility register the programmer can use to store some data, to be later used in calculations, writing to RAM or MMIO, etc. The main use of this register is as an offset from some fixed address. For example, a programmer could write a simple loop to iterate over every element in a certain address range, with very few lines of code.

- Y - Exactly the same as X, although some instructions use Y instead of X and vise versa.

- A (short for Accumulator) - We've already seen that one the section above. A 1 byte register used by the accumulator to output the result of a calculation. The programmer can also use this register.

- PC (short for Program Counter) - This register holds the data for the next instruction to be executed. After each instruction the counter gets incremented some amount to point to the next instruction. The incrementation amount differs depending on the instruction and its addressing mode, as different instructions and addressing modes accommodate different amounts of space (For example in the example above the `lda` instruction only used 1 operand, which is 1 byte and the `sta` instruction used 2 operands, which make up to be 2 bytes)

- S (short for Stack Pointer) - This register holds the address from which the stack starts. The stack grows downwards

- P (status register) - This registers bits are used as flags to keep track of certain stuff an instruction done. For example, if an `inc` caused A to overflow out of its bounds and caused a reset. If interrupts to the CPU are disabled, if the current instruction caused a value to become negative, etc. For more information, check out [this page](https://www.nesdev.org/wiki/Status_flags)

## The clock

The clock is the component of the CPU which tells it what it should do now. Each clock tick the CPU moves to the next step of the fetch-decode-execute cycle.

## Interrupts

Say the user pressed a button, or another component of the machine has finished doing something and needs the CPU, or some other important and urgent event that happened, and the CPU should respond to. How does it do that?
Well, one of the ways is using interrupts.
interrupts are essentially signals sent to the CPU from other components, outside the CPU which causes the CPU to stop the instruction its currently doing, and respond to the new event that just happened.

Interrupts could be very costly, as they make the CPU drop everything its doing, save what it was doing (so later when it finished handling the event it knowns where it stopped off), and take care of the event.
So interrupts should used only when really necessary.

The 6502 has 2 different kinds of interrupts:

- IRQ (Interrupt request) - A kind of interrupt used by mappers for example. This pin is mostly used for lower priority events, as the CPU can ignore them by setting the Interrupt Disable flag in the P register.
It pushes the P and PC registers onto the stack, and writes the IRQ interrupt vector (which is on addresses `$fffe` and `$ffff`) to PC

- NMI (Non maskable interrupt) - As the name implies, this is an interrupt the CPU cannot ignore. It is used for more urgent and important events. A use of this interrupt is the PPU for example, when it enters the first vblank scanline (we will talk about it in the PPU internals write up)
It pushes the P and PC registers onto the stack, and writes the NMI interrupt vector (which is on addresses `$fffa` and `$fffb`) to PC

**Note:**

I'm writing this clarification, because I remember it made me confused:
Some people say there is another interrupt, called BRK (or software interrupt). This is not entirely true. BRK is an instruction of the 6502, which lets the programmer generate an interrupt from the game/programs code, but in reality, all it does is cause an IRQ interrupt, with the special addition of setting the `B` flag on the P register to 1.

## Final words

Hope this writeup helped understanding how the CPU of the NES works. I know I say this every writeup, but even though it is very hard to understand as a beginner, I can't recommend the NESdev wiki enough. It contains everything emu devs or game devs of the NES need to know
