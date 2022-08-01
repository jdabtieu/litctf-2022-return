# addition
![](https://img.shields.io/badge/category-reversing-blue)
![](https://img.shields.io/badge/solves-111-orange)

## Description
Enter the flag, and it will be checked with addition!

[addition](https://drive.google.com/uc?export=download&id=1hRfIzZrNdkjxLpUtnc_8uwyd2jYpOvhV)

## Solution
Opening the file in Ghidra, we can see that it is stripped. Not an issue though, and after some cleaning up, a fairly readable main function emerges.

![](https://cdn.discordapp.com/attachments/359503958601105418/1003398229758578708/unknown.png)

There are two key arrays, and to check if the input is correct, it treats the machine code of `main` as an array, and adds `main[key1[i]]` to `key2[i]` and compares that to the user input.

Instead of trying to figure out the values ourselves, we can run the program in GDB and pause right before each comparison to see what the current letter of the flag should be.

The binary is stripped, so we'll have to slowly find our way to main. Run `starti` and then keep running `ni` until the current instruction is `call   QWORD PTR [rip+0x2ea6]`. The guessed argument contains the address of main. The next instruction to be run is `si`, to step into main.

![](https://cdn.discordapp.com/attachments/359503958601105418/1003403818014359783/unknown.png)

We can then disassemble main by running `gdb-peda$ x/50i 0x5592871a9070` (replace `0x5592871a9070` with whatever address is printed in the previous output).

Comparing this to Ghidra's decompilation, I can see that these two instructions are responsible for checking the flag.
```asm
   0x5592871a90c5:      cmp    edx,esi
   0x5592871a90c7:      je     0x5592871a90a8
```
To leak the flag character by character, we can set a breakpoint at `0x5592871a90c5` (address of the compare instruction) and every time we hit the breakpoint, we can inspect the value of the `edx` register, and then force jump to `0x5592871a90a8`.
```gdb
gdb-peda$ b *0x5592871a90c5
Breakpoint 1 at 0x5592871a90c5
gdb-peda$ c
Continuing.
123456789012345678901234
[----------------------------------registers-----------------------------------]
RAX: 0x0 
RBX: 0x7ffeb218b940 ("123456789012345678901234")
RCX: 0x5592871ac0c0 --> 0x5e0000004c ('L')
RDX: 0x4c ('L')
... [ output omitted ] ...
```
Great, `rdx` (64-bit version of `edx` contains `L`, which we know is the first character of the flag.
```gdb
gdb-peda$ jump *0x5592871a90a8
Continuing at 0x5592871a90a8.
[----------------------------------registers-----------------------------------]
RAX: 0x1 
RBX: 0x7ffeb218b940 ("123456789012345678901234")
RCX: 0x5592871ac0c0 --> 0x490000004c ('L')
RDX: 0x49 ('I')
... [ output omitted ] ...
```
Continuing to issue the jump command (bypassing the jump-if-equal), we can leak the entire flag.

Flag: `LITCTF{add1ti0n_is_h4rd}`
