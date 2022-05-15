---
title: Binary Multiplicative Inverse
date: 2022-05-15
---

Given `uint64_t y = x * 0xDEADBEEFCAFEF00D`, could you calculate `x` when `y = 0x3644C87C4F3391E8`?

Your first thought may be to calculate `0x3644C87C4F3391E8 / 0xDEADBEEFCAFEF00D`, but that will just give an answer of 0.

You could also just use [z3](https://github.com/Z3Prover/z3), but that's no fun.
```py
import z3
z3.solve(z3.BitVec('x', 64) * 0xDEADBEEFCAFEF00D == 0x3644C87C4F3391E8)
```

Instead, let's choose a simpler number and go back to basics.

## Binary Multiplication

Lets say you want to calculate `uint8_t y = x * 0x45`, but without using multiplication.
If you inspect the binary representation of 0x45, you get:
```py
>>> bin(0x45)
'0b1000101'
```
That is to say: `0x45 == 0b1 + 0b100 + 0b1000000`.

Now if you remember from Algebra that `(a + b) * c = (a * c) + (b * c)`, you could instead write `uint8_t y = x * 0x45` as: `uint8_t y = (x * 0b1) + (x * 0b100) + (x * 0b1000000)`.
That still uses multiplications, but since each of those numbers are powers of two, they can be replaced with bit-shifts: `uint8_t y = (x << 0) + (x << 2) + (x << 6)`.

Why is this useful? Well if you think back to the original question, what you really want to find is a way to remove all those extra bit-shifted copies of the original number that were added together during the multiplication.
For `0x45`, that means getting rid of the `(x << 2) + (x << 6)`.

## Carry, carry, carry!

To get rid of those extra copies, you can keep carrying them into higher bits (until they have all overflowed), by adding more copies of the original number you multipled by. For example:
```c
uint8_t a = 0b01000101; // 0x45
uint8_t b = a;

assert(b == 0b01000101);
b += a << 2; // Carry the 0b00000100

assert(b == 0b01011001);
b += a << 3; // Carry the 0b00001000

assert(b == 0b10000001);
b += a << 7; // Carry the 0b10000000

assert(b == 0b00000001);
```

This means the multiplicative inverse of `0x45` is `(1 << 0) + (1 << 2) + (1 << 3) + (1 << 7)`, or `0x8D`.

```cpp
uint8_t a = 0x1F;
uint8_t b = a * 0x45;
assert(a == (uint8_t)(b * 0x8D));
```

## 0xDEADBEEFCAFEF00D

So, how do you do this for `0xDEADBEEFCAFEF00D`?
Well, lets write a loop to which iterates over each of the unwanted bits and carries them to a higher bit, until they have all overflowed.
```c
uint64_t a = 0xDEADBEEFCAFEF00D;
uint64_t b = a;
uint64_t c = 1;

// Iterate over all the bits of b, apart from the lowest bit
// (which represents the original "copy" of the number)
for (size_t i = 1; i < 64; ++i) {
    // The bit we want to check
    uint64_t bit = UINT64_C(1) << i;
    
    // Is this bit set?
    if (b & bit) {
        // Eliminate this bit by carrying it into a higher bit
        b += a << i;

        // Keep track of the bits we carried
        c |= bit;
    }
}

printf("0x%016" PRIX64, c); // 0xA761C9B0BCBEDEC5
```

Running this outputs `0xA761C9B0BCBEDEC5`, and indeed checking it with the original question gives the correct result: `0x1122334455667788`
```c
uint64_t y = 0x3644C87C4F3391E8;
uint64_t x = y * 0xA761C9B0BCBEDEC5;
assert(y == (uint64_t)(x * 0xDEADBEEFCAFEF00D));
printf("0x%016" PRIX64 "\n", x); // 0x1122334455667788
```

However, it is important to note: because `b += a << i` assumes the lowest bit of `a` is set, this will only work when the constant you are multiply by is odd.

## Conclusion
While the [modular multiplicative inverse](https://en.wikipedia.org/wiki/Modular_multiplicative_inverse) isn't something unique to binary numbers, thinking about it in binary can provide some useful insights into how binary multiplication works.