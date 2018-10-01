---
layout: post
title:  "Emulating Op-Codes"
image: ''
date:   2018-10-01 17:00:00
tags:
- emulation
- mos6502
- java
- functional
description: 'A look into the evolution of how my Java emulator -EmuRox- emulated 6502 op-codes'
categories:
- Java
- Emulation 
---

## Introduction

The largest personal project I am currently undertaking, is that of an [emulator](https://github.com/rossdrew/emuRox).  The main goal was to come up with a larger product that I am completely in control of and can experiment with tools and methods to become a better programmer.  Sub goals include learning something about the machines I grew up with and maybe, perhaps getting to the point where I've written my own retro NES emulator.  Long term, pie in the sky goal was to create a very nicely designed, highly tested, pluggable, multi-emulator.
I wanted to strongly focus on what I feel is important in a Java program:

 - Highly Tested
 - Readable
 - Efficient
 - Maintainable
 
Satusfying all of the above id equal -IMHO- to "well designed".

I started with the most documented part of the NES.  The MOS6502 processor.  

## Approach Number 1: The Result of Test Driven Development (TDD)

One of the approaches I've been playing with is TDD.  Where I write a failing test, then write the code to make it work and iteratively build a fully tested, modular product.  This led me to writing the minimal amount of code to make tests work.  The result was great process and...this:

```
    public void step() {
        log.debug("STEP >>>");

        final OpCode opCode = OpCode.from(nextProgramByte().getRawValue());

        //Execute the opcode
        log.debug("Instruction: {}...", opCode.getOpCodeName());
        switch (opCode){
            case ASL_A:
                withRegister(Register.ACCUMULATOR, this::performASL);
            break;

            case ASL_Z:
                withByteAt(RoxWord.from(nextProgramByte()), this::performASL);
            break;

            case ASL_Z_IX:
                withByteXIndexedAt(RoxWord.from(nextProgramByte()), this::performASL);
            break;

            case ASL_ABS_IX:
                withByteXIndexedAt(nextProgramWord(), this::performASL);
            break;

            case ASL_ABS:
                withByteAt(nextProgramWord(), this::performASL);
            break;

            case LSR_A:
                withRegister(Register.ACCUMULATOR, this::performLSR);
            break;

            case LSR_Z:
                withByteAt(RoxWord.from(nextProgramByte()), this::performLSR);
            break;

            case LSR_Z_IX:
                withByteXIndexedAt(RoxWord.from(nextProgramByte()), this::performLSR);
            break;

            case LSR_ABS:
                withByteAt(nextProgramWord(), this::performLSR);
            break;

            case LSR_ABS_IX:
                withByteXIndexedAt(nextProgramWord(), this::performLSR);
            break;

            ...
```

This is of course truncated as there are many, many more opcodes on the 6502.  Let's look at whether it fits my criteria

 - Highly Tested:  _Floats around 98-99% line coverage.  Static analyis. Mutation tested to 98%_
 - Readable: Partially. Each opcode has a path.  The instructions to execute it are in english. Little annoying that you need to scroll so much and complicated instructions end up being messy to nest method calls so that they read fluidly.
 - Efficient: For large numbers of cases, switch statements are pretty efficient.
 - Maintainable: No.  There are loads of methods that are there to reduce duplication which end up being duplications themselves.  Parts of the logic are no seperable like addressing and operations.  

So I wanted to see if I could do better.

### The legacy & and the beauty of TDD

It led to some pretty hard to maintain code, but correct; and we have tests to prove it.  Which means we can mess with other implementations and the huge test space will theoretically catch any mistakes.

## Approach Number 2: Functional Java Code

I want to minimise duplication, writing code once for each operation, and once for each addressing mode.  As we already reference opcodes by Enum, I wondered if it was possible to parse an opcode value (`0x0A`) into it's Enum (`ASL_A`) then simply call an `execute()` method on that Enum instance which fires off an attached lambda providing an environment (memory, registers and alu).  As each Op-Code Enum knows it's own Addressing Mode, it could call an `address()` on that Addressing Mode (which also calls an attached lambda) providing us with all we need to make composable instructions.

### Overview

So my enum becomes a definition, a byte value, it's addressing mode and the operation performed.  The OpCode is then the intersection of Addressing Mode and Operation

```
ASL_A(0x0A, AddressingMode.ACCUMULATOR, Operation.ASL);  
```

The operation is an operation on the environment, given an addressed value:

```
    public enum Operation implements AddressedValueInstruction {
        /** Shift all bits in byte left by one place, setting flags based on the result */
        ASL((a,r,m,v) -> {
            final RoxByte newValue = a.asl(v);
            r.setFlagsBasedOn(newValue);
            return newValue;
        }),

        ...

        @Override
        public RoxByte perform(Mos6502Alu alu, Registers registers, Memory memory, RoxByte value) {
            return instruction.perform(alu, registers, memory, value);
        }
```

The addressing mode uses an environment to address a value, runs a given operation on it and places the return value back at the addressed location:

```
public enum AddressingMode implements Addressable {
    /** Expects no argument, operation will be performed using the Accumulator Register*/
    ACCUMULATOR("Accumulator", 1, (r, m, a, i) -> {
        final RoxByte value = r.getRegister(Registers.Register.ACCUMULATOR);
        r.setRegister(Registers.Register.ACCUMULATOR, i.perform(a, r, m, value));
    }),

    ...

    @Override
    public void address(Registers r, Memory m, Mos6502Alu alu, AddressedValueInstruction instruction) {
        address.address(r, m, alu, instruction);
    }

```

Then the op-code brings it all together by combining the addressing mode and operation:

```
public enum OpCode implements Instruction {
    
    ...

    @Override
    public void perform(Mos6502Alu alu, Registers registers, Memory memory) {
        addressingMode.address(registers, memory, alu, operation::perform);
    }
```

This means the huge `switch` statement becomes one line of code

```
opCode.perform(alu, registers, memory);
```

### Problems

There are some instructions which don't fit this pattern nicely:

  - `AddressingMode.IMPLIED`: The addressing mode is implied by the last two bits of the instruction byte

The `IMPLIED` instructions do slightly different addressing based on the `Operation`.  I'll need to deal with these as individual cases, where `Operation`s do their own addressing.

  - `Operation.JMP`: It -unlike all other instructions- passes a word (not a byte) from the addressing mode that could be `AddressingMode.INDIRECT`

These do two things that make them hard to deal with.  Firstly, their addressing is `ABSOLUTE`; the two byte (word) address that the `Operation` is supposed to use to load into the Program Counter.  We could deal with this in the same way as the `IMPLIED` (in that the `Operation` then does some of it's own addressing) if not for the case where `JMP` uses `INDIRECT`-`ABSOLUTE` addressing, which will take a two byte argument then from the address specified by that word, load a two byte address into the Program Counter.  In this case, it cannot be done in the `Operation` because it has no idea what it's Addressing Mode is.

