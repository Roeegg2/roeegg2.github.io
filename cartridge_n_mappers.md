# The Cartridge & Mappers

## Introduction

If you haven't checked out my PPU and CPU writeups, I recommend you go check them out before reading this part, as I'm building on top of things I explained there.

## Terminology

1. ROM - a type of memory (which unlike RAM) its data cannot change (this is why its also called ROM - "read only memory". as you can only read data from it). The data in it is set once by the manufacturer, and cannot be changed again by the CPU.

## What is a cartridge?

Modern consoles have full operating systems, and some firmware to boot the operating system (like BIOS or UEFI).
The NES doesn't have any of that - it uses cartridges.
These are essentially just memory chips (usually ROM, but sometimes RAM) which contains the data of the game.
The cartridge is made up of (in order):

### Number 0 - header information 

I marked this as 0 since real cartridges the real NES used don't actually have this information, but in order for an emulator to run the game it should be provided with some additional information (such as which mapper is used, the size of the CHR and PRG roms, is RAM or ROM used, etc)

### Number 1 - PRG

A ROM chip containing the _code_ of the game

### Number 2 - CHR 

A ROM chip containing the _graphics_ data (AKA the pattern tables!) of the game

### Number 3 - Interrupt Vectors

Pretty self explanatory, go check out the CPU writeup if you don't know what these are.
`Reset` interrupt vector resides in
`IRQ` interrupt vector resides in 
`NMI` interrupt vector resides in

When the CPU boots, it performs a 'reset' - meaning it jumps to the address stored at the _reset vector_ (the _reset vector_ is stored at `$fffc` and `$fffd`). That data at `$fffc` and `$fffd` is the address of the _reset procedure_ written by the game programmers, which starts the game.

### Conclusion

```
SOME NOTES:
1. Some cartridges use CHR RAM, so that the game actually changes the contents of the cartridge during gameplay, and not just the nametables
2. Some cartridges also have a PRG RAM chip with a battery for dumping the CPU memory (address range `$0000-$0800` in CPU memory map) when exiting the game, so progress is saved.
```

So overall, pretty simple - just the raw code & data, and a small section which specifies where are the ISRs located.

## What is a mapper?

If we go check the [memory map of the CPU](https://www.nesdev.org/wiki/CPU_memory_map), we see that the memory map of the CPU is 16 bits, in the range `$0000-$ffff` - which gives us `($10000 - $4020) / 1024 = (64-16)KB = 48KB` of data.
Similarly for the PPU, the range is `$0000-$1fff` - which gives us `$2000 / 1024 = 8KB` of data.

This is little amount of data. VERY little amount.
So how does the NES display some pretty good looking games with many levels such as Super Mario Bros 3, Kirby, Legend Of Zelda, etc?

The answer is.........

Mappers! (I bet that one caught you off guard :)

A Mapper is essentially a small circuit that acts as a "middle man" between the NES components, and the cartridge data we just discussed (PRG, CHR, interrupt vectors).

It lets the game developers create games that use more than just the limited `48KB` and `8KB` CPU and PPU memory maps, by exposing some ports (registers) which select a different "part" of the data.
These "parts" the cartridge data is divided into are called _banks_.

And so by selecting different banks, the NES components can access more data.

There are a lot of mappers out there (too many if you ask me). Different mappers supports a different number of banks, expose a different set of registers, and have a different mechanism for changing and setting the values of their bank selection registers.

```
SOME NOTES:

1. Not every mapper exposes bank selection registers - NROM (iNES mapper number 0) provides no "extra" banks. It supports just the basic NES memory maps, so there is no need for bank selection registers - the same banks are used all the time. 

2. Remember I said there are a lot of mappers? Well..... It's kind of true. There are indeed a lot of mappers, but many of them are based off other mappers with some very minor and small modification.
The main mappers used, and the ones which most other mappers are based off are: NROM, UNROM, CNROM, MMC1, and MMC3.
```
