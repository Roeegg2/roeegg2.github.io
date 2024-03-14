# PPU sprites

## OAM

The OAM (object attribute memory) is the place where sprites are stored. It has the size of 256 bytes, and can store 64 sprites. 
Using a short calculation, we get that each sprite entry is the size of `256/64 = 4 Bytes`.

There is another memory area called secondary OAM, which can hold up to 8 sprites (32 bytes).
Data is written to OAM, and then transfered over to secondary OAM, from which the sprites are actually drawn

### Byte 0

This byte contains the Y position of the sprite, counted from the top of the screen.
The value here is actually subtracted by 1, to get the actual Y value of the sprite. I will explain the reasoning behind this later on.

### Byte 1

This byte contains the tile number to be used, to index into the pattern tables.
Its use differs between 8x16 and 8x8:

From the background graphics chapter, we already know we know the PPU can address 2 pattern tables (`$0000-$0fff` and `$1000-$1fff`).
For 8x8 sprites, the decision of which pattern table to use is given by bit 3 of PPUCTRL.
For 8x16 sprites, bit 0 of the this byte, is used to select the pattern table.

### Byte 2

This byte contains some important attribute data about the sprite: palette group used, sprite priority, and whether the sprite should be rendered flipped (both vertically or horizontally)

### Byte 3

The X position of the sprite, counted from the left of the sprite (ie the first pixel)


Thats it! That's how sprites are represented in the NES.


## DMA

Remember I said the PPU copies data from primary OAM into secondary OAM?
Well the PPU has 2 options to perform these copies:
1. Copying using OAMADDR and OAMDATA
2. Copying using DMA

For those who dont know, a DMA is a circuit which its purpose is to handle writes and reads from MMIO from the CPU, so the CPU can do other things in the meanwhile.
In the case of the NES, the CPU is actually halted while the DMA is copying data to secondary OAM, so it doesn't have that advantage, but the copy is actually way faster using DMA (it takes 2 cycles to copy a byte using DMA, and the CPU has overhead of resolving the addressing mode, fetching the next instruction, etc. The DMA only knows how to copy from primary OAM to secondary OAM so it doesnt have as much overhead)

The DMA 


This is memory section which is internal to the PPU.

<---"just thought about it, and isnt it kinda weird the DMA section of the PPU is called DMA?
isnt the whole point of a DMA is having another circuit handle I\O for the CPU, thus while a transfer is being made the CPU can execute other stuff in the meanwhile?">