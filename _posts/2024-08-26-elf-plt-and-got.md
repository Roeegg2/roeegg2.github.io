---
layout: post
title: The PLT + GOT, Lazy binding + Eager binding  
date: 2024-08-25 15:13:25 +0300
permalink: /elf-plt-and-got
author: Roee Toledano
---

# Prologue

This is just a short post explaining the GOT and PLT in the ELF format. No bullshit, just explanation.

Any assembly here will be x86_64, Intel syntax, but it's the same jist for different architectures as well.

# PLT + GOT explained in a nutshell

The PLT, GOT and GOT.PLT are all sections in an ELF file.

- The PLT is a table of entries, each entry being a very short code snippet (often times called _stub_).
- The GOT is also a table of entries, each entry being a an address to some symbol (could be function, global variable, etc).
- The GOT.PLT is simply the part of the GOT which the PLT uses.

Usually when we call a function, we use `call <address of function we want to call>`. 
> **_NOTE:_** the operand for `call` doesn't have to be a hardcoded address. It might be the offset from `RIP`, maybe we use the value in some register as address, etc but either way it's the same gist - we call the function directly.
After the instruction is executed the CPU pushes `RIP` to the stack and jumps to the specified address.

Over all simple.

Well, it's not always the case.
Calling functions which utilize the PLT looks a bit different:

1. instead of `call`ing straight to the address of the function we want to execute, we `call` to the address of some PLT entry.
2. the PLT entry, contains a `jmp` instruction to _another_ address - a GOT.PLT entry
3. the GOT.PLT entry contains the actual address of the function we wanted to call.

> **_NOTE:_** as you'll see for yourself very soon, usually the PLT stub has additional code, not just a single `jmp` instruction to the desired GOT.PLT entry. We'll get to that in a bit.

# Motivation

The first time I've come across this mechanism the first thing I asked myself "Isn't this just overcomplicating things?" you might ask.
Well, at first glance it does seem a bit complicated, but like everything that gets implemented (ok, maybe not _everything_) there are good reasons for it:

1. In case a relocation must be applied (for example when the function is implemented by a shared object) instead of applying relocations straight to the `.text` section in every place the function is called, you apply it once to an entry in the GOT entry, and you're good to go!
2. Using different versions of functions - sometimes, the developer wants to call a different function depending on what version, architecture, OS, etc
the user is rocking. That way the dynamic linker can choose which address to replace the GOT with, depending on what function should be called (this is what happens in a lot of glibc functions, like `printf` for example)
3. Lets the linker use a technique called _lazy binding_.

Wha'ts lazy binding you ask? Well, let me tell you!

# Lazy vs eager binding in a nutshell

Lazy binding and eager binding are different methods to apply relocations (binding function calls to their actual address)

In **eager** binding, the linker resolves the relocations before transfaring control to the program. In **lazy** binding, (as the name implies) the linker does this lazily - he doesn't initially resolve the relocations, but instead it resolves a relocation at runtime, when the program tries to use the symbol associated with that relocation.

The reason lazy binding was introduced was because in large applications, applying all relocations before running could be very time consuming, which would result in quite a big delay until the program actually runs. On the other hand, applying relocations at runtime results in a decrease of total execution speed. 

Glibc uses lazy binding by default unless specified otherwise, but in contrast Google's Fuchsia libc uses eager binding.

There isn't one truth, and each method is better for different scenarios depending on the programs priority and requirements.

# Running example

Let's see all of this in action now!
Consider the following C program:

```c
/* prog.c */
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int* ptr = malloc(10 * sizeof(int));
    for (int i = 0; i < 10; i++) {
        ptr[i] = i;
        printf("%d\n", ptr[i]);
    }

    return 0;
}
```
A simple trivial program, which allocates memory for 10 integers, moves over the array and prints the values.
> **_NOTE:_** It is bad practice to call `malloc` without checking if the pointer is `NULL`, and it is also bad practice to not `free` the memory after you're done with it. This is just an example, so I'm not doing any of that. But don't be like me - always check for errors and free your memory!

Compiling this with `gcc -o prog prog.c`
> **_NOTE:_** I'm running all of this on a `x86_64 Arch Linux` machine (`uname -r = 6.11.3-arch1-1`). Things might be a bit different on your machine, but the general idea is the same. Make sure glibc is dynamically loaded and not statically linked.

When running the program, we get as expected:
```bash
[roeet@roeetarch ~]$ ./prog
0
1
2
3
4
5
6
7
8
9

```


Running `objdump -d prog` we get inter alia:
```asm
Disassembly of section .plt:

0000000000001020 <printf@plt-0x10>:
    1020:       ff 35 ca 2f 00 00       push   0x2fca(%rip)        # 3ff0 <_GLOBAL_OFFSET_TABLE_+0x8>
    1026:       ff 25 cc 2f 00 00       jmp    *0x2fcc(%rip)        # 3ff8 <_GLOBAL_OFFSET_TABLE_+0x10>
    102c:       0f 1f 40 00             nopl   0x0(%rax)

0000000000001030 <printf@plt>:
    1030:       ff 25 ca 2f 00 00       jmp    *0x2fca(%rip)        # 4000 <printf@GLIBC_2.2.5>
    1036:       68 00 00 00 00          push   $0x0
    103b:       e9 e0 ff ff ff          jmp    1020 <_init+0x20>

0000000000001040 <malloc@plt>:
    1040:       ff 25 c2 2f 00 00       jmp    *0x2fc2(%rip)        # 4008 <malloc@GLIBC_2.2.5>
    1046:       68 01 00 00 00          push   $0x1
    104b:       e9 d0 ff ff ff          jmp    1020 <_init+0x20>
```

