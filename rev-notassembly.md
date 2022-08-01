# not assembly
![](https://img.shields.io/badge/category-reversing-blue)
![](https://img.shields.io/badge/solves-73-orange)

## Description
Find the output and wrap it in `LITCTF{}`!

[notassembly.png](https://drive.google.com/uc?export=download&id=1mV8gjChjRDSLuFUYDS_1lZemxA-SBNa7)

## Solution
While the code isn't an obvious assembly variant (x64, ARM, etc.) as the name suggests, Googling some of its unique instructions such as `bl`, `bu`, and `print` reveal that it is the [ASCL assembly language](https://www.categories.acsl.org/wiki/index.php?title=Assembly_Language_Programming).

Using the details on the page, it is relatively simply to emulate this program in Python or any other language, including other flavors of Assembly. There is also [an interpreter](https://github.com/lachm/acsl.git) that can run the program for us.

```py
flag = 0
codetiger = 23
orz = 4138
acc = orz
while True:
  acc = (acc * 3) % 1000000
  acc = (acc + codetiger) % 1000000
  flag = acc
  acc -= 9538
  if acc < 0:
    break

print(flag - 3571)
```

Flag: `LITCTF{5149}`
