# save_tyger
![](https://img.shields.io/badge/category-pwn-blue)
![](https://img.shields.io/badge/solves-166-orange)

## Description
Can you save our one and only Tyger?

Connect with `nc litctf.live 31786`

[save_tyger.zip](https://drive.google.com/uc?export=download&id=1ePTPwUBKcNLESM2ev1IZEb3kL4SeTcn1)

## Solution
In `save_tyger.c`, we can see that the objective is to overwrite `pass` with `0xabadaaab`.

Because pass is located above buf in the program, its location in memory will be right after buf.

Also, because `gets(buf)` uses the dangerous gets function which allows us to write past the end of buf, we can use it to overwrite pass, by first sending garbage to fill up buf, and then writing onto pass.

It's important to note that because all integer types are stored as little-endian, we should be writing `\xab\xaa\xad\xab` (backwards).

To determine how much garbage to write, we know that buf has a size of 32, and pass comes right after it, but the stack might be padded to be 16-byte aligned.

Thus, the offset should be either 32 or 40 bytes. We can try both of these:
```
$ python -c 'import sys; sys.stdout.buffer.write(b"A"*32 + b"\xab\xaa\xad\xab\n")' | nc litctf.live 31786
$ python -c 'import sys; sys.stdout.buffer.write(b"A"*40 + b"\xab\xaa\xad\xab\n")' | nc litctf.live 31786
```
The second one works, and gives the flag. It's required to add a newline at the end of our input to terminate `gets`.

Flag: `LITCTF{y4yy_y0u_sav3d_0ur_m41n_or94n1z3r}`
