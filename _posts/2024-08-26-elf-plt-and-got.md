---
layout: post
title: The PLT and the GOT
date: 2024-08-25 15:13:25 +0300
permalink: /elf-plt-and-got
author: Roee Toledano
---

# Prologue

This is just a short post explaining the GOT and PLT in the ELF format. No bullshit, just explanation.

Any assembly here will be x86_64, Intel syntax, but it's the same jist for different architectures as well.

# Everything explained in a nutshell

The PLT, GOT and GOT.PLT are all sections in an ELF file.

- The PLT is a table of entries, each entry being a very short code snippet (often times called _stub_).
- The GOT is also a table of entries, each entry being a an address to some symbol (could be function, global variable, etc).
- The GOT.PLT is a sub-table the GOT table. The PLT's GOT.

When we call a function, we usually use `call <address of function we want to call>`.
After the instruction is executed the CPU pushes `RIP` to the stack and jumps to the specified address.

Over all simple.

Well, it's not always the case.
Calling functions which utilize the PLT looks a bit different:

1. instead of `call`ing straight to the address of the function we want to execute, we `call` to the address of some PLT entry.
2. the PLT entry, contains a `jmp` instruction to _another_ address - a GOT.PLT entry
3. the GOT.PLT entry contains the actual address of the function we wanted to call.


# Motivation

"Why go through all of that shit?" you might ask.
Well, a few reasons:

1. In case a relocation must be applied (for example when the function is implemented by a shared object) Instead of applying relocations straight to the `.text`
section in every place the function is called, you apply it once to an entry in the GOT entry, and you're good to go!
2. Using different versions of functions - sometimes, the developer wants to call different function depending on what version, architecture, OS, etc
the user is rocking. That way the dynamic linker can choose which address to replace the GOT with, depending on what function should be called (this is what
happens in a lot of glibc functions, like `printf` for example)
3. Lets the linker use a technique called _lazy binding_[^1].

# Running example

Let's see all of this in action now!
We'll take as an example the famous `ls` command.

<!-- CONTINUE HERE -->

# The dynamic linkers process of handling this

In this section (pun intended :), we'll learn exactly what the dynamic linker does "behind the curtains" to make this cool mechanism work

We'll continue with our `ls` example.
Let's `readelf -a /bin/ls | less` to see the parsed details of the file.
> **_NOTE:_** Output might be a different for you, depending on arch, OS etc. I'm doing all of this on a `Linux fedora 6.10.5-200.fc40.x86_64` machine.
(You can get this info yourself by running `uname` in terminal)

We get a bunch of stuff. These are the juicy parts we're interested in:
.dynamic entries
```
Dynamic section at offset 0x219a8 contains 31 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libselinux.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libcap.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000c (INIT)               0x3000
 0x000000000000000d (FINI)               0x18fa4
 0x0000000000000019 (INIT_ARRAY)         0x21e50
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x21e58
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x458
 0x0000000000000005 (STRTAB)             0x10e0
 0x0000000000000006 (SYMTAB)             0x498
 0x000000000000000a (STRSZ)              1620 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x22be8
 0x0000000000000002 (PLTRELSZ)           2664 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x1a90
 0x0000000000000007 (RELA)               0x1940
 0x0000000000000008 (RELASZ)             336 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000000000001e (FLAGS)              BIND_NOW
 0x000000006ffffffb (FLAGS_1)            Flags: NOW PIE
 0x000000006ffffffe (VERNEED)            0x1840
 0x000000006fffffff (VERNEEDNUM)         2
 0x000000006ffffff0 (VERSYM)             0x1734
 0x0000000000000024 (RELR)               0x24f8
 0x0000000000000023 (RELRSZ)             72 (bytes)
 0x0000000000000025 (RELRENT)            8 (bytes)
 0x0000000000000000 (NULL)               0x0
```

More specifically, we care about these entries:
```
0x0000000000000003 (PLTGOT)             0x22be8 // offset from the start of the file where the .got.plt is located at
0x0000000000000002 (PLTRELSZ)           2664 (bytes)  // total size of the PLT section
0x0000000000000014 (PLTREL)             RELA // type of relocations entries in the PLT.RELA table[^2]
0x0000000000000017 (JMPREL)             0x1a90 // offset from the start of the file where the .plt.rela section located at (yes, weird naming, I know)
```

<!-- CONTINUE HERE -->

<!-- footnotes -->
