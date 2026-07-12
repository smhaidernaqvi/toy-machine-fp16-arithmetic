# Floating-Point Arithmetic on the TOY Machine

This project implements **floating-point addition, subtraction, multiplication,
and division** on the **TOY machine** (a simple 16-bit hypothetical computer,
taught in Princeton's COS 126) using only its integer instruction set —
`ADD`, `SUB`, `AND`, `XOR`, `SHIFT`, `LOAD`, `STORE`, and `BRANCH`.
No hardware floating-point unit is used; every operation is built entirely
from these primitives.

---

## 1. Floating-Point Representation (IEEE-754-style)

Real IEEE 754 (the standard used by every modern CPU/FPU) represents a
number as:

```
value = (-1)^sign  ×  1.mantissa  ×  2^(exponent - bias)
```

It splits a word into three fields:

| Field | Purpose |
|---|---|
| **Sign bit** | `0` = positive, `1` = negative |
| **Exponent** | stored with a **bias** added, so it can represent both positive and negative powers of 2 using only unsigned bits |
| **Mantissa (fraction)** | the digits after the binary point of a *normalized* number `1.xxxxx` |

**Implicit vs. explicit mantissa bit:** A normalized binary number always
looks like `1.xxxxx × 2^e` — the leading `1` is *guaranteed* to be there
(except for the special case of zero/denormals), so IEEE 754 doesn't
bother storing it. This is the **hidden / implicit bit**. Only the
fractional part `xxxxx` (the *explicit* mantissa) is physically stored in
the word. Before doing arithmetic, hardware must **restore** this hidden
`1` bit; after arithmetic, it must be **stripped** again before packing
the result back into the word.

**Bias:** The exponent field is stored as an unsigned number, but exponents
can be negative (for numbers < 1). So a constant **bias** is added when
storing, and subtracted when reading. For IEEE-754 single precision the
bias is 127; for our smaller custom format, it's 15.

### Our custom 16-bit format

Since a real 32-bit IEEE-754 float needs a 32-bit ALU, we use a
**scaled-down 16-bit version** of the same idea, sized to fit the TOY
machine's 16-bit registers:

```
 bit:  15   14 13 12 11 10    9 8 7 6 5 4 3 2 1 0
      [ S ][    EXPONENT    ][      MANTISSA       ]
        1         5                    10
```

- **1 sign bit**
- **5-bit exponent**, bias = **15** (`0x000F`)
- **10-bit mantissa** (explicit fraction bits only — the leading `1.` is
  implicit, exactly like real IEEE 754)

Example: `4.25 = 1.0001₂ × 2²` → sign `0`, exponent `2+15=17=0x11`,
mantissa `0001000000` → packed word = `0x4440`.

---

## 2. The 5 Core Steps of Floating-Point Arithmetic

Every operation below (add, subtract, multiply, divide) follows the same
five-stage pipeline — this is exactly what a real FPU's arithmetic
pipeline does in hardware:

| # | Step | What happens |
|---|---|---|
| **1** | **Extraction** | Split the 16-bit word into sign, exponent, and mantissa using `AND` masks and `SHIFT`. Restore the implicit hidden bit (`+ 0x0400`) to get the true 11-bit significand. |
| **2** | **Alignment** | Only needed for add/subtract. The two exponents must match before their mantissas can be added/subtracted directly — the smaller-exponent operand's mantissa is right-shifted until the exponents are equal (this is why floating-point addition isn't "just" integer addition). |
| **3** | **Arithmetic** | The actual operation on the significands: `ADD` (same-sign add), `SUB` (different-sign add / subtract), repeated shift-and-add (multiply), or restoring shift-and-subtract (divide). |
| **4** | **Normalization** | The raw result may not be in the `1.xxxxx` form anymore — addition can overflow (needs a right-shift + exponent++), subtraction can cause leading-bit cancellation (needs a left-shift loop + exponent--). This step restores the "implicit 1" invariant. |
| **5** | **Packing** | Strip the hidden bit back out, shift the exponent into position, `OR`/`ADD` the sign bit back in (disjoint bit-fields, so `ADD` works exactly like `OR` here), and `STORE` the final 16-bit word. |

---

## 4. Addition & Subtraction — `addition.toy`

### Pseudocode

```
1. EXTRACTION
   signA, signB   = A[15], B[15]
   expA,  expB    = (A & ExpMask)  >> 10 , (B & ExpMask)  >> 10
   mantA, mantB   = (A & MantMask) + Hidden , (B & MantMask) + Hidden

2. ALIGNMENT
   diff = expA - expB
   if diff == 0:      no shift needed
   if diff > 0:        mantB >>= diff ; expB += diff
   if diff < 0:        mantA >>= |diff| ; expA += |diff|
   (now expA == expB)

3. ARITHMETIC  (sign-aware!)
   if signA == signB:                       # true addition
       result = mantA + mantB
       resultSign = signA
   else:                                     # true subtraction
       if mantA >= mantB:
           result = mantA - mantB ; resultSign = signA
       else:
           result = mantB - mantA ; resultSign = signB

4. NORMALIZATION
   if signA == signB and result overflowed (bit 11 set):
       result >>= 1 ; exponent += 1
   if signA != signB:                        # possible cancellation
       while result != 0 and bit10(result) == 0:
           result <<= 1 ; exponent -= 1
       if result == 0: pack a true zero and stop

5. PACKING
   result = strip_hidden_bit(result)
   Result = resultSign<<15 | exponent<<10 | result
```

### Why the same code handles subtraction too

`A − B` is mathematically identical to `A + (−B)`. Since our arithmetic
step (step 3 above) is **already sign-aware** — it checks whether the
signs match or differ and reacts accordingly — subtraction needs **zero
new instructions**. To compute `A − B`, simply flip **B's sign bit**
(`bit 15`, i.e. `XOR B with 0x8000`) before loading it into memory, and
run the exact same `addition.toy` program. This was verified directly:

| Operation | A (hex) | B (hex, sign flipped) | Result (hex) |
|---|---|---|---|
| `4.25 − 2.12` | `4440` | `C03D` | `4044` |
| `2.12 − 4.25` | `403D` | `C440` | `C044` |
| `4.25 − 4.25` | `4440` | `C440` | `0000` |
| `-4.25 − 2.12` | `C440` | `C03D` | `C65E` |

### Machine code

```
00: 0000   01: 4440   02: 403D   03: 0000
04: 000F   05: 7C00   06: 03FF   07: 0400
08: 000A   09: 0001   0A: FFFF   0B: 0800
0C: 0000   0D: 0000   0E: 0000   0F: 0000
10: 8101   11: 8202   12: 8304   13: 6413
14: 6523   15: 8605   16: 3716   17: 3826
18: 8908   19: 6779   1A: 6889   1B: 8A06
1C: 3B1A   1D: 3C2A   1E: 8D07   1F: 1BBD
20: 1CCD   21: 2178   22: C12D   23: D12B
24: 8E0A   25: 411E   26: 8209   27: 1112
28: 6BB1   29: 1771   2A: C02D   2B: 6CC1
2C: 1881   2D: 4645   2E: C63F   2F: 21BC
30: D135   31: C135   32: 21CB   33: 1F50
34: C036   35: 1F40   36: 7301   37: C13D
38: 321D   39: D248   3A: 5113   3B: 2773
3C: C037   3D: 9003   3E: C04F   3F: 11BC
40: 820B   41: 3221   42: C246   43: 7301
44: 1773   45: 6113   46: 1F40   47: C048
48: 411D   49: 5779   4A: 4117   4B: 760F
4C: 5FF6   4D: 111F   4E: 9103   4F: 91FF
50: 0000
```

### Test cases (addition)

| A | B | Expected | Result (decimal) | Result (hex) |
|---|---|---|---|---|
| 4.25 | 2.12 | 6.37 | 6.3672 | `465E` |
| -4.25 | 2.12 | -2.13 | -2.1328 | `C044` |
| 1.0 | -4.25 | -3.25 | -3.2500 | `C280` |
| 4.25 | -4.25 | 0.0 | 0.0000 | `0000` |
| -4.25 | -2.12 | -6.37 | -6.3672 | `C65E` |

### Test cases (subtraction, same program, B's sign flipped)

| A | B | Expected | Result (decimal) | Result (hex) |
|---|---|---|---|---|
| 4.25 | 2.12 | 2.13 | 2.1328 | `4044` |
| 2.12 | 4.25 | -2.13 | -2.1328 | `C044` |
| 4.25 | 4.25 | 0.0 | 0.0000 | `0000` |
| -4.25 | 2.12 | -6.37 | -6.3672 | `C65E` |

*(Small differences like 6.37 vs. 6.3672 are expected rounding error —
2.12 cannot be represented exactly in a 10-bit mantissa, so the nearest
representable value, 2.119140625, is used instead.)*

---

## 5. Multiplication — `multiply.toy`

### Pseudocode

```
1. EXTRACTION
   signA, signB   = A[15], B[15]
   expA,  expB    = (A & ExpMask) >> 10 , (B & ExpMask) >> 10
   mantA, mantB   = (A & MantMask) + Hidden , (B & MantMask) + Hidden

2. (no alignment needed for multiplication)

3. ARITHMETIC
   resultSign = signA XOR signB
   resultExp  = expA + expB - bias          # exponents ADD, bias only once
   
   # TOY has no MUL instruction -> sequential shift-and-add multiply
   HI = 0
   LO = mantB
   repeat 11 times:
       if LSB(LO) == 1:  HI = HI + mantA
       carry = LSB(HI)
       HI >>= 1
       LO >>= 1
       LO = LO | (carry << 15)      # combined 32-bit right shift of (HI:LO)
   candidate = (HI << 1) + (LO >> 15)     # top ~12 bits of the 22-bit product

4. NORMALIZATION
   if bit11(candidate) set:            # product landed in [2,4) instead of [1,2)
       candidate >>= 1 ; resultExp += 1

5. PACKING
   candidate = strip_hidden_bit(candidate)
   Result = resultSign<<15 | resultExp<<10 | candidate
```

**Why shift-and-add works:** TOY has no multiply circuit, so
multiplication is built the way early/simple CPUs do it in hardware — a
*sequential multiplier*. For each bit of one operand (LSB first), if the
bit is `1`, the other operand is added into a running accumulator; then
the accumulator and the shrinking multiplier are shifted right together
as if they were one 32-bit register (`HI:LO`). After 11 iterations
(matching the 11-bit significand width), `HI:LO` together hold the full
22-bit product.

### Machine code

```
00: 0000   01: 4440   02: 403D   03: 0000
04: 000F   05: 7C00   06: 03FF   07: 0400
08: 000A   09: 0001   0A: FFFF   0B: 0800
0C: 0000   0D: 0000   0E: 0000   0F: 0000
10: 8101   11: 8202   12: 8304   13: 6413
14: 6523   15: 4F45   16: 8605   17: 3716
18: 3826   19: 8908   1A: 6779   1B: 6889
1C: 1778   1D: 2773   1E: 8A06   1F: 3B1A
20: 3C2A   21: 8D07   22: 1BBD   23: 1CCD
24: 1E00   25: 7201   26: 760B   27: C632
28: 38C2   29: C82B   2A: 1EEB   2B: 38E2
2C: 6EE2   2D: 6CC2   2E: 5883   2F: 1CC8
30: 2662   31: C027   32: 5EE2   33: 68C3
34: 1EE8   35: 860B   36: 38E6   37: C83A
38: 6EE2   39: 1772   3A: 4EED   3B: 5779
3C: 1EE7   3D: 5FF3   3E: 1EEF   3F: 9E03
40: 9EFF   41: 0000
```

### Test cases

| A | B | Expected | Result (decimal) | Result (hex) |
|---|---|---|---|---|
| 4.25 | 2.12 | 9.01 | 9.00 | `4880` |
| 2.0 | 3.0 | 6.0 | 6.0 | `4600` |
| 1.5 | 1.5 | 2.25 | 2.25 | `4080` |
| -4.25 | 2.12 | -9.01 | -9.00 | `C880` |
| -2.0 | -3.0 | 6.0 | 6.0 | `4600` |
| 1.0 | 1.0 | 1.0 | 1.0 | `3C00` |

---

## 6. Division — `division.toy`

### Pseudocode

```
1. EXTRACTION
   signA, signB   = A[15], B[15]
   expA,  expB    = (A & ExpMask) >> 10 , (B & ExpMask) >> 10
   mantA, mantB   = (A & MantMask) + Hidden , (B & MantMask) + Hidden

2. (no alignment needed for division)

3. ARITHMETIC
   resultSign = signA XOR signB
   resultExp  = expA - expB + bias           # exponents SUBTRACT

   # ensure a valid starting point for restoring division:
   if mantB > mantA:
       mantA <<= 1 ; resultExp -= 1          # scale A up so mantA/mantB stays in range

   # TOY has no DIVIDE instruction -> restoring (shift-subtract) division
   remainder = mantA - mantB
   quotient  = 0
   repeat 10 times:
       remainder <<= 1
       quotient  <<= 1
       if remainder >= mantB:
           remainder -= mantB
           quotient  |= 1

4. NORMALIZATION
   quotient = quotient & MantMask        # keep the low 10 bits as the mantissa field
                                          # (the implicit leading "1" needs no extra step —
                                          # it's the IEEE-style hidden bit, never stored)

5. PACKING
   Result = resultSign<<15 | resultExp<<10 | quotient
```

**Why restoring division works:** just like long division on paper,
each iteration doubles the remainder (brings down the next bit),
compares it against the divisor, and records a `1` or `0` quotient bit
depending on whether the subtraction "fits". Because TOY significands
can only be in the range `[1024, 2047]` (i.e. `mantA / mantB` always
lands in `(0.5, 2.0)`), a special case is needed: if `mantB > mantA`,
`mantA` is pre-scaled by `×2` (with a compensating `exponent -= 1`) so
the very first subtraction is guaranteed non-negative before the
restoring-division loop begins.

### Machine code

```
00: 0000   01: 4440   02: 403D   03: 0000
04: 03FF   05: 0400   06: 7C00   07: 000F
08: 000A   09: 0001   0A: 0000   0B: 0000
0C: 0000   0D: 0000   0E: 0000   0F: 0000
10: 8101   11: 8202   12: 8907   13: 6319
14: 6429   15: 8906   16: 3519   17: 3629
18: 8908   19: 6559   1A: 6669   1B: 8904
1C: 3719   1D: 3829   1E: 8905   1F: 1779
20: 1889   21: 4934   22: 2A56   23: 8E07
24: 1AAE   25: 2F87   26: DF29   27: 2B78
28: C02E   29: 8E09   2A: 5B7E   2B: 2BB8
2C: 8E09   2D: 2AAE   2E: 8E09   2F: 1C0E
30: 8D08   31: 8E09   32: 5BBE   33: 5CCE
34: 2F8B   35: DF39   36: 2BB8   37: 8E09
38: 1CCE   39: 8E09   3A: 2DDE   3B: DD31
3C: 8E04   3D: 3CCE   3E: 8E08   3F: 5AAE
40: 8E07   41: 599E   42: 11AC   43: 1119
44: 9103   45: 91FF   46: 0000
```

### Test cases

| A | B | Expected | Result (decimal) | Result (hex) |
|---|---|---|---|---|
| 4.25 | 2.12 | 2.0055 | 2.0039 | `4002` |
| 2.0 | 3.0 | 0.6667 | 0.6665 | `3955` |
| 1.0 | 1.5 | 0.6667 | 0.6665 | `3955` |
| -4.25 | 2.12 | -2.0047 | -2.0039 | `C002` |
| 4.25 | -2.12 | -2.0047 | -2.0039 | `C002` |
| -4.25 | -2.12 | 2.0047 | 2.0039 | `4002` |

*(Small rounding differences come from truncation rather than
round-to-nearest — the restoring-division loop takes the floor of the
quotient bits, not the rounded value.)*

---

## 7. Instruction Set Reference

| Opcode | Mnemonic | Format | Effect |
|---|---|---|---|
| 0 | halt | — | stop execution |
| 1 | add | d s t | R[d] = R[s] + R[t] |
| 2 | sub | d s t | R[d] = R[s] - R[t] |
| 3 | and | d s t | R[d] = R[s] & R[t] |
| 4 | xor | d s t | R[d] = R[s] ^ R[t] |
| 5 | lshift | d s t | R[d] = R[s] << R[t] |
| 6 | rshift | d s t | R[d] = R[s] >> R[t] |
| 7 | loadaddr | d addr | R[d] = addr (immediate) |
| 8 | load | d addr | R[d] = M[addr] |
| 9 | store | d addr | M[addr] = R[d] |
| A | loadind | d addr | R[d] = M[M[addr]] |
| B | storeind | d addr | M[M[addr]] = R[d] |
| C | branchz | d addr | if R[d]==0: PC = addr |
| D | branchp | d addr | if R[d]>0: PC = addr |
| E | jumpr | d | PC = R[d] |
| F | jumpl | d addr | R[d] = PC+1; PC = addr |
