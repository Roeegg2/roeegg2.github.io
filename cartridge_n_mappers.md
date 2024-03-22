# The Cartridge & Mappers

## Introduction

If you haven't checked out my PPU and CPU writeups, I recommend you go check them out before reading this part, as I'm building on top of things I explained there.

## Terminology

1. ROM - a type of memory (which unlike RAM) its data cannot change (this is why its also called ROM - "read only memory". as you can only read data from it). The data is written to it once by the manufacturer, and cannot be changed again by the CPU.

## What is a cartridge?

Modern consoles have full operating systems, some firmware to boot the operating system. (like BIOS or UEFI)
The NES doesn't have any of that - it uses cartridges.
These are essentially just memory chips (usually ROM, but sometimes RAM) which contains (in order):

0. (header information - in brackets since real ROMs don't have these, but modern .nes files have a header which contains important information for the emulator (like which mapper is used, the size of the CHR and PRG roms, is RAM or ROM used, etc))
1. the code (often called PRG data) of the game
2. the graphics (often called CHR data) data of the game
3. interrupt vectors (recall from the CPU writeup)

When the CPU boots, it performs a 'reset' - meaning it jumps to the address stored at the _reset vector_ (the _reset vector_ is stored at `$fffc` and `$fffd`). That data at `$fffc` and `$fffd` is the address of the _reset procedure_ written by the game programmers, which starts the game.

So overall, pretty simple - just the raw code & data, and a small section which specifies where are the ISRs located.

## What is a mapper?

If we go check the [memory map of the CPU](https://www.nesdev.org/wiki/CPU_memory_map), we see that the memory map of the CPU is 16 bits, in the range `$0000-$ffff` - which gives us `($10000 - $4020) / 1024 = (64-16)KB = 48KB` of data.
Similalrly for the PPU, the range is `$0000-$1fff` - which gives us `$2000 / 1024 = 8KB` of data.

This is little amount of data. VERY little amount.
So how does the NES display some pretty good looking games with many levels such as Super Mario Bros 3, Kirby, Legend Of Zelda, etc?
