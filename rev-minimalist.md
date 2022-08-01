# minimalist
![](https://img.shields.io/badge/category-reversing-blue)
![](https://img.shields.io/badge/solves-53-orange)

## Description
less is more

[minimalist](https://drive.google.com/uc?export=download&id=1vMY6FRx_Eff2ypd9vaZCRr6HYNdPsneX)

## Solution
Opening the binary in Ghidra, it's immediately obvious that it's stripped. Cleaning up the main function helps a bit, showing the input phase of the program:

![](https://user-images.githubusercontent.com/62577178/182051971-aa5e861f-5da1-4961-a2c2-33529265e85f.png)

For some reason Ghidra is not recognizing stack variables correctly, but essentially, this is what happens:
1. reads a char
2. if it's the 0-th char, we push it onto the stack
3. grab the i-th value of the "key1" array and push it onto the stack
4. grab the i-th value of the "key2" array and push it onto the stack
5. push the current character onto the stack

At the end of this loop of 47 iterations, the stack should contain:
```
[rep 47 times] char[i]
[rep 47 times] key2[i]
[rep 47 times] key1[i]
               char[0]
```

The next major segment of code still looks like a disaster, and the disassembly looks like hand-written assembly, as it only consists of `or`, `not`, `push`, and `pop`.

![](https://user-images.githubusercontent.com/62577178/182052083-2890cc95-8262-4ac8-985e-a1209255f451.png)

Trying to read through it had me instantly disorientated, and I knew there was no easy way of understanding what it was trying to do. So instead, I decided to work backwards. I could see that `answer_bool` was only `or`-d with other stuff, and that it had to equal 0 at the end. Thus, anything it's `or`-d with must be 0 as well.

![](https://user-images.githubusercontent.com/62577178/182052335-01e2d38b-6980-4eea-af55-58f540cf528d.png)

`0010128c`: Starting at the end, `$rax` overwrites `answer_bool`. At the top, we can see that `$rax` is `answer_bool`.<br>
`00101289`: `$rax` is `or`d with `$r12` on the previous line, so `$r12` must also be 0.<br>
`00101286`: `$r12` is set to `not` itself, so it should have a value of `!0`, or `0xffffffffffffffff`<br>
`00101283`: `$r12` is `or`d with `$r10`, and the result must have all 64 bits set.<br>
`00101281`: `$r10` and `$r12` are popped from the stack. The corresponding<br>
`0010127f`: pushed values are `$r14` and `$r12` respectively, from `00101266` and `00101264`.<br>
`0010127c`: a different `$r12` is `or`d with `$rax`, so `$r12` must be 0.<br>
`00101279`: `$r12` is set to `not` itself, so it should have a value of `!0`, or `0xffffffffffffffff`<br>
`00101276`: `$r12` is `or`d with `$r10`, and the result must have all 64 bits set.<br>
`00101274`: `$r11` is copied to `$r12`<br>
`00101272`: See previous instruction<br>
`0010126f`: `$r11` is inverted<br>
`0010126c`: `$r10` is inverted<br>
`0010126a`: pushed values are `$r14` and `$r12` respectively, from `00101266` and `00101264`<br>
`00101268`: This part ensures that `$r14` and `$r12` have no overlapping bits.<br>
`00101268`: Combined with the check from `00101276`, this whole thing is just `$r12 ^ $r14 == -1`

This is because all 64 bits must be set by combining the two registers, but they must also not have overlapping bits, so `xor`ing the two registers would have all 64 bits set.

At this point, the objective is clear, so instead of figuring out the rest of the Assembly mess, I decided to just emulate the other instructions and brute-force check if the xor condition held or not.

There is one last thing to check though: what happens with the stack?

![](https://user-images.githubusercontent.com/62577178/182053238-2f6618f7-902f-4268-90dc-ce2704f10700.png)

Running though the `POP` and `PUSH` instructions, we can see that:
1. current char is popped off the stack
2. current key2 is popped off the stack
3. current key1 is popped off the stack
4. previous char is popped off the stack
5. current char is restored to the stack
6. every other `PUSH`/`POP` is balanced.

So actually, it looks like the last character (`}`) is always on the stack, and key1 and key2 actually operate on the *previous* char, except for the first character, because the first character is copied twice. However, this is irrelevant if we start brute forcing from the first character of the flag, instead of going down the stack.

Because Python integers do not behave like C `long`s in terms of bitwise operations (as it's not bound to 64 bits), I decided to use Java's shell instead. After extracting the two key arrays from Ghidra, I used the following code to brute force the flag:
```java
long[] key2 = {52, 96, 122, 30, 57, 75, 121, 37, 88, 20, 43, 72, 117, 86, 51, 99, 104, 125, 16, 20, 2, 63, 99, 127, 100, 123, 13, 5, 112, 58, 125, 96, 12, 47, 41, 76, 8, 65, 119, 31, 27, 97, 83, 53, 120, 53, 63, 0};

long[] key1 = {0xffffffffffffff87L, 0xffffffffffffffd3L, 0xffffffffffffffccL, 0xffffffffffffffb5L, 0xffffffffffffff85L, 0xffffffffffffffe0L, 0xffffffffffffffc0L, 0xffffffffffffffa1L, 0xfffffffffffffff0L, 0xffffffffffffff83L, 0xffffffffffffffe4L, 0xffffffffffffffe8L, 0xffffffffffffffe4L, 0xffffffffffffff9aL, 0xffffffffffffffffL, 0xfffffffffffffff8L, 0xffffffffffffffe4L, 0xffffffffffffffddL, 0xffffffffffffff8eL, 0xffffffffffffffdaL, 0xffffffffffffffccL, 0xffffffffffffff9fL, 0xffffffffffffffe8L, 0xffffffffffffffe8L, 0xffffffffffffffabL, 0xfffffffffffffff7L, 0xffffffffffffffb7L, 0xffffffffffffffa5L, 0xffffffffffffffe9L, 0xfffffffffffffff1L, 0xffffffffffffffecL, 0xfffffffffffffffcL, 0xffffffffffffff8aL, 0xffffffffffffff8fL, 0xffffffffffffffe7L, 0xffffffffffffffddL, 0xffffffffffffff84L, 0xffffffffffffffcaL, 0xfffffffffffffffaL, 0xffffffffffffff95L, 0xffffffffffffff87L, 0xffffffffffffffeaL, 0xffffffffffffffc5L, 0xffffffffffffffa5L, 0xffffffffffffffe9L, 0xffffffffffffffb9L, 0xffffffffffffffffL};

String res = "";
for (int i = 0; i < 47; i++) {
 for (char j = ' '; j < 127; j++) {
  System.out.println(res + j);
  long r13 = '}';
  long r11 = key2[i];
  long r14 = key1[i];
  long r10 = j;
  r10 = ~r10;
  long r12 = r11;
  r12 |= r10;
  r12 = ~r12;
  r10 = ~r10;
  r11 = ~r11;
  r13 = r11;
  r13 |= r10;
  r13 = ~r13;
  r12 |= r13;
  if ((r12 ^ r14) == -1) {
   res += j;
   break;
  }
 }
}
```

Essentially, it loops through every character and performs the first set of instructions until the xor, then checks if the xor result is correct. If so, it appends the guessed character to the flag, and starts working on the next character.

The final guess printed is `LLITCTF{Wh0_n33ds_a11_th0sE_f4ncy_1nstructions?`, and the duplicated `L` at the front makes sense, as the first character was duplicated.

It's also missing the final closing bracket because the last character is not checked in this loop, simply popped and pushed repeatedly on the stack. Fixing these two issues by hand, we get the flag.

Flag: `LITCTF{Wh0_n33ds_a11_th0sE_f4ncy_1nstructions?}`
