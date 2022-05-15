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
If you inspect the binary representation of `0x45`, you get:
```py
>>> bin(0x45)
'0b1000101'
```
That is to say: `0x45 == 0b1 + 0b100 + 0b1000000`.

Now if you remember from algebra that `(a + b) * c = (a * c) + (b * c)`, you could instead write `uint8_t y = x * 0x45` as: `uint8_t y = (x * 0b1) + (x * 0b100) + (x * 0b1000000)`.
That still uses multiplications, but since each of those numbers are powers of two, they can be replaced with bit-shifts: `uint8_t y = (x << 0) + (x << 2) + (x << 6)`.

Why is this useful? Well if you think back to the original question, what you really want to find is a way to remove all those extra bit-shifted copies of the original number that were added together during the multiplication.

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

        assert(b == (uint64_t)(a * c));
        printf("0x%016" PRIX64 " " "0x%016" PRIX64 "\n", b, c);
    }
}

assert(b == 1);
printf("The inverse of 0x%016" PRIX64 " is " "0x%016" PRIX64 "\n", a, c);
```

Running this outputs:
```
0x5964BAAEF6FAB041 0x0000000000000005
0x04D476A1B6B6B381 0x0000000000000045
0x5BB3EE87362EBA01 0x00000000000000C5
0xB731CE1D340ED401 0x00000000000002C5
0x6E2D8D492FCF0801 0x00000000000006C5
0xDC250BA1274F7001 0x0000000000000EC5
0xB814085116504001 0x0000000000001EC5
0x27CFFB10D2538001 0x0000000000005EC5
0x0747E0904A5A0001 0x000000000000DEC5
0x8527768E2A740001 0x000000000002DEC5
0x80E6A289EAA80001 0x000000000006DEC5
0x7864FA816B100001 0x00000000000EDEC5
0x6761AA706BE00001 0x00000000001EDEC5
0x455B0A4E6D800001 0x00000000003EDEC5
0xBD4089C674000001 0x0000000000BEDEC5
0x7C6C8586A8000001 0x0000000004BEDEC5
0xFAC47D0710000001 0x000000000CBEDEC5
0xF7746C07E0000001 0x000000001CBEDEC5
0xF0D44A0980000001 0x000000003CBEDEC5
0xD653C21000000001 0x00000000BCBEDEC5
0x8642C2E000000001 0x00000010BCBEDEC5
0xE620C48000000001 0x00000030BCBEDEC5
0x6598CB0000000001 0x000000B0BCBEDEC5
0x6488D80000000001 0x000001B0BCBEDEC5
0x5C09400000000001 0x000009B0BCBEDEC5
0x180C800000000001 0x000049B0BCBEDEC5
0x9013000000000001 0x0000C9B0BCBEDEC5
0x8020000000000001 0x0001C9B0BCBEDEC5
0x81C0000000000001 0x0021C9B0BCBEDEC5
0x8500000000000001 0x0061C9B0BCBEDEC5
0x9200000000000001 0x0161C9B0BCBEDEC5
0xAC00000000000001 0x0361C9B0BCBEDEC5
0xE000000000000001 0x0761C9B0BCBEDEC5
0x8000000000000001 0x2761C9B0BCBEDEC5
0x0000000000000001 0xA761C9B0BCBEDEC5
The inverse of 0xDEADBEEFCAFEF00D is 0xA761C9B0BCBEDEC5
```
And indeed, using it to solve the original question gives the correct answer: `0x1122334455667788`
```c
uint64_t y = 0x3644C87C4F3391E8;
uint64_t x = y * 0xA761C9B0BCBEDEC5;
assert(y == (uint64_t)(x * 0xDEADBEEFCAFEF00D));
printf("0x%016" PRIX64 "\n", x); // 0x1122334455667788
```

However, it is important to note: because `b += a << i` assumes the lowest bit of `a` is set, this will only work when the constant you are multiplying by is odd (has the lowest bit set).

## Conclusion
While the [modular multiplicative inverse](https://en.wikipedia.org/wiki/Modular_multiplicative_inverse) isn't something unique to binary numbers, thinking about it in binary can provide some useful insights into how binary multiplication works.