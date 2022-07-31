# math test
![](https://img.shields.io/badge/category-reversing-blue)
![](https://img.shields.io/badge/solves-176-orange)

## Description
this math test is hard

<https://drive.google.com/uc?export=download&id=1jGE3v40Xk3-Fq2GsnGvwzU8prZEoL3Iz>

## Solution
Opening the binary in Ghidra, the main function appears to print 10 questions, read the user answers, and then call the `grade_test` function.

Cleaning up the `grade_test` function, since we know that there are 10 questions, and some unknown block of memory with size 80 must be a `long long[10]` or its unsigned variant, we can get the following:

![](https://cdn.discordapp.com/attachments/359503958601105418/1003387466818003044/unknown.png)

It seems like the answers are stored directly in the program, we just have to read them.

Double clicking `answers` to jump to its location in the binary, we can select the array, right click, and copy special as Python byte string:
```
b'\x02\x00\x00\x00\x00\x00\x00\x00\x04\x00\x00\x00\x00\x00\x00\x00\xf0\x00\x00\x00\x00\x00\x00\x00\x03\x00\x00\x00\x00\x00\x00\x00\x6d\x8d\xde\x09\x00\x00\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x87\x15\x59\x00\x00\x00\x00\x00\x49\x1e\xa1\x06\x00\x00\x00\x00\x5f\xe9\x60\x20\x00\x00\x00\x00\x09\x00\x00\x00\x00\x00\x00\x00'
```

The array is in little-endian, so we can use pwntools to help us convert it into decimal.
```py
from pwn import *
x = b'\x02\x00\x00\x00\x00\x00\x00\x00\x04\x00\x00\x00\x00\x00\x00\x00\xf0\x00\x00\x00\x00\x00\x00\x00\x03\x00\x00\x00\x00\x00\x00\x00\x6d\x8d\xde\x09\x00\x00\x00\x00\x0a\x00\x00\x00\x00\x00\x00\x00\x87\x15\x59\x00\x00\x00\x00\x00\x49\x1e\xa1\x06\x00\x00\x00\x00\x5f\xe9\x60\x20\x00\x00\x00\x00\x09\x00\x00\x00\x00\x00\x00\x00'
ans = [u64(x[i*8:i*8+8]) for i in range(10)]
print(ans)
# [2, 4, 240, 3, 165580141, 10, 5838215, 111222345, 543222111, 9]
```

Finally, we can run the program with these answers, to get the flag.

Flag: `LITCTF{y0u_must_b3_gr8_@_m4th_i_th0ught_th4t_t3st_was_imp0ss1bl3!}`
