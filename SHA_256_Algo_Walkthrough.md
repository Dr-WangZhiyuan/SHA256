# SHA-256 Algo Complete Walkthrough 

**Input: `abcd`**

**Target output:** `88d4266fd4e6338d13b845fcf289579d209c897823b9217da3e161936f031589`

---

## Overview

SHA-256 takes any input, processes it in 512-bit blocks through 64 rounds of bit-mixing, and outputs a fixed 256-bit (32-byte) digest. The same input always produces the same output; any change in input completely changes the output.

---

## Step 1: Binary Conversion

Convert each ASCII character to its 8-bit binary representation:

| Char | Decimal | Binary     | Hex  |
|------|---------|-----------|------|
| `a`  | 97      | `01100001` | 0x61 |
| `b`  | 98      | `01100010` | 0x62 |
| `c`  | 99      | `01100011` | 0x63 |
| `d`  | 100     | `01100100` | 0x64 |

Concatenated as one binary string (32 bits total):

```
01100001 01100010 01100011 01100100
```

**Original message length L = 4 × 8 = 32 bits.**

---

## Step 2: Message Padding (to 512 bits)

SHA-256 processes data in 512-bit blocks. The padded message must satisfy:

```
[original message] [1 bit] [zero bits] [64-bit length]
= exactly 512 bits (or a multiple of 512)
```

**Padding calculation:**

```
512 bits total
-  32 bits  (original message)
-  64 bits  (length field at the end)
= 416 bits of padding needed

= 1 bit (the mandatory '1') + 415 zero bits
```

**Padded message (512 bits = 64 bytes):**

The 512-bit padded message is split into 16 primary 32-bit blocks

```
Block:   W0
Bits:    01100001011000100110001101100100

Block:   W1
Bits:    10000000000000000000000000000000

Block:   W2..W14
Bits:    00000000000000000000000000000000

Block:   W15
Bits:    00000000000000000000000000100000


W[0]  = 0x61626364  ← "abcd" in ASCII hex
W[1]  = 0x80000000  ← the '1' padding bit, followed by zeros
W[2] to W[14] = 0x00000000  ← all zeros
W[15] = 0x00000020  ← original length (32) as 64-bit big-endian = 0x0000000000000020
```

**Why pad this way?**  

The `1` bit marks where the real message ends. The 64-bit length at the end allows SHA-256 to process messages of different sizes unambiguously.

---

## Step 3: Message Schedule (W₀ to W₆₃)

The 512-bit block gives us 16 words (W[0]..W[15]). We expand to 64 words using:

$$W_t = \sigma_1(W_{t-2}) + W_{t-7} + \sigma_0(W_{t-15}) + W_{t-16} \pmod{2^{32}} \quad \text{for } t = 16 \text{ to } 63$$

**First 16 words (from padded message):**

```
W[ 0] = 0x61626364   (binary: 01100001011000100110001101100100)
W[ 1] = 0x80000000   (binary: 10000000000000000000000000000000)
W[ 2] = 0x00000000
W[ 3] = 0x00000000
W[ 4] = 0x00000000
W[ 5] = 0x00000000
W[ 6] = 0x00000000
W[ 7] = 0x00000000
W[ 8] = 0x00000000
W[ 9] = 0x00000000
W[10] = 0x00000000
W[11] = 0x00000000
W[12] = 0x00000000
W[13] = 0x00000000
W[14] = 0x00000000
W[15] = 0x00000020   (binary: 00000000000000000000000000100000 = 32 decimal)
```

**Example expansion — W[16]:**

```
W[16] = σ₁(W[14]) + W[9] + σ₀(W[1]) + W[0]  (mod 2³²)

σ₁(W[14]) = σ₁(0x00000000) = 0x00000000
W[9]       = 0x00000000
σ₀(W[1])  = σ₀(0x80000000) = ROTR⁷(0x80000000) ⊕ ROTR¹⁸(0x80000000) ⊕ SHR³(0x80000000)
           = 0x01000000 ⊕ 0x00002000 ⊕ 0x10000000
           = 0x11002000
W[0]       = 0x61626364

W[16]      = 0x00000000 + 0x00000000 + 0x11002000 + 0x61626364 = 0x72628364
```

**First 5 expanded words:**

```
W[16] = 0x72628364
W[17] = 0x80140000
W[18] = 0x11c22fdd
W[19] = 0x80205508
W[20] = 0x52115a52
```

---

## Step 4: Helper Functions

SHA-256 uses six core functions, all operating on 32-bit words:

### Lowercase σ (used in message schedule)

| Function | Formula | Purpose |
|----------|---------|---------|
| σ₀(x) | ROTR⁷(x) ⊕ ROTR¹⁸(x) ⊕ SHR³(x)   | Mixes message words for schedule |
| σ₁(x) | ROTR¹⁷(x) ⊕ ROTR¹⁹(x) ⊕ SHR¹⁰(x)  | Mixes message words for schedule |

### Uppercase Σ (used in compression)

| Function | Formula | Purpose |
|----------|---------|---------|
| Σ₀(x) | ROTR²(x) ⊕ ROTR¹³(x) ⊕ ROTR²²(x) | Mixes working variable `a` |
| Σ₁(x) | ROTR⁶(x) ⊕ ROTR¹¹(x) ⊕ ROTR²⁵(x) | Mixes working variable `e` |

### Non-linear functions

**Ch (Choose):** x selects bit-by-bit from y (where x=1) or z (where x=0)
```
Ch(x, y, z) = (x ∧ y) ⊕ (¬x ∧ z)
```

**Maj (Majority):** output bit = whichever value (0 or 1) appears in majority among x, y, z
```
Maj(x, y, z) = (x ∧ y) ⊕ (x ∧ z) ⊕ (y ∧ z)
```

### Bitwise operations

| Operation | Symbol | Meaning |
|-----------|--------|---------|
| Right Rotate n | ROTR^n | Shift right n bits; bits falling off the right wrap to the left |
| Right Shift n  | SHR^n  | Shift right n bits; zeros fill from the left (bits are lost) |
| XOR           | ⊕      | 1 if bits differ, 0 if same |
| AND           | ∧      | 1 only if both bits are 1 |
| NOT           | ¬      | Flip all bits |

---

## Step 5: Initial Hash Values (H₀ to H₇)

Derived from **fractional parts of square roots of the first 8 primes**, multiplied by 2³²:

**Example for prime 2:**
```
√2          = 1.41421356237...
Fractional  = 0.41421356237...
 × 2³²      = 1,779,033,703
In hex      = 0x6a09e667
```

**All 8 initial values:**

```
H[0] = 0x6a09e667   (from √2)
H[1] = 0xbb67ae85   (from √3)
H[2] = 0x3c6ef372   (from √5)
H[3] = 0xa54ff53a   (from √7)
H[4] = 0x510e527f   (from √11)
H[5] = 0x9b05688c   (from √13)
H[6] = 0x1f83d9ab   (from √17)
H[7] = 0x5be0cd19   (from √19)
```

**Why these constants?** "Nothing up my sleeve" numbers — derived from a verifiable formula, proving no hidden backdoor was designed in.

---

## Step 6: Round Constants (K₀ to K₆₃)

Derived from **fractional parts of cube roots of the first 64 primes**, multiplied by 2³²:

**Example for prime 2:**
```
∛2          = 1.25992104989...
Fractional  = 0.25992104989...
 × 2³²      = 1,116,352,408
In hex      = 0x428a2f98
```

**First 8 of 64 round constants:**

```
K[ 0] = 0x428a2f98   K[ 1] = 0x71374491   K[ 2] = 0xb5c0fbcf   K[ 3] = 0xe9b5dba5
K[ 4] = 0x3956c25b   K[ 5] = 0x59f111f1   K[ 6] = 0x923f82a4   K[ 7] = 0xab1c5ed5
```

---

## Step 7: Compression (64 Rounds)

### Initialize 8 working variables from H:

```
a = H[0] = 0x6a09e667
b = H[1] = 0xbb67ae85
c = H[2] = 0x3c6ef372
d = H[3] = 0xa54ff53a
e = H[4] = 0x510e527f
f = H[5] = 0x9b05688c
g = H[6] = 0x1f83d9ab
h = H[7] = 0x5be0cd19
```

### Each round formula (i = 0 to 63):

repeat this sequence 64 times so that every word $W_0$…$W_{63}$ and constant $K_0$…$K_{63}$ mixes thoroughly into the working state.

```
T₁ = h + Σ₁(e) + Ch(e,f,g) + K[i] + W[i]    (mod 2³²)
T₂ = Σ₀(a) + Maj(a,b,c)                      (mod 2³²)

h = g
g = f
f = e
e = d + T₁    (mod 2³²)
d = c
c = b
b = a
a = T₁ + T₂  (mod 2³²)
```

Think of a, b, c, d, e, f, g, h as a **shift register** — each round, everything shifts one position, and two new values enter at `a` and `e`.

### Round 0 (i=0) detailed example:

```
Inputs:
  h = 0x5be0cd19
  e = 0x510e527f,  f = 0x9b05688c,  g = 0x1f83d9ab
  a = 0x6a09e667,  b = 0xbb67ae85,  c = 0x3c6ef372
  K[0] = 0x428a2f98
  W[0] = 0x61626364

Compute:
  Σ₁(e)    = Σ₁(0x510e527f) = ROTR⁶(e) ⊕ ROTR¹¹(e) ⊕ ROTR²⁵(e) = 0xb95e669d
  Ch(e,f,g) = 0x9b05688c (where e=1 take f, where e=0 take g)     = 0x9b41e8ac
  T₁ = 0x5be0cd19 + 0xb95e669d + 0x9b41e8ac + 0x428a2f98 + 0x61626364 = (mod 2³²) → feeds into e and a

After Round 0:
  a = 0x5d6aebb1
  b = 0x6a09e667
  c = 0xbb67ae85
  d = 0x3c6ef372
  e = 0xfa2a4606
  f = 0x510e527f
  g = 0x9b05688c
  h = 0x1f83d9ab
```

### After Round 1:

```
  a = 0x4cc9fb6a,  b = 0x5d6aebb1,  c = 0x6a09e667,  d = 0xbb67ae85
  e = 0x6c4e6b69,  f = 0xfa2a4606,  g = 0x510e527f,  h = 0x9b05688c
```

### After Round 2:

```
  a = 0x75515850,  b = 0x4cc9fb6a,  c = 0x5d6aebb1,  d = 0x6a09e667
  e = 0x74e169cc,  f = 0x6c4e6b69,  g = 0xfa2a4606,  h = 0x510e527f
```

*(Rounds 3–63 continue the same pattern with W[3]..W[63] and K[3]..K[63])*

---

## Step 8: Final Hash Computation

After all 64 rounds, add the final working variables back to the initial H values (mod 2³²):

```
H[0] = H[0] + a  (mod 2³²)
H[1] = H[1] + b  (mod 2³²)
H[2] = H[2] + c  (mod 2³²)
H[3] = H[3] + d  (mod 2³²)
H[4] = H[4] + e  (mod 2³²)
H[5] = H[5] + f  (mod 2³²)
H[6] = H[6] + g  (mod 2³²)
H[7] = H[7] + h  (mod 2³²)
```

Concatenate H[0]..H[7] as hex strings:

```
88d4266f  d4e6338d  13b845fc  f289579d  209c8978  23b9217d  a3e16193  6f031589
```

**Final SHA-256("abcd") = `88d4266fd4e6338d13b845fcf289579d209c897823b9217da3e161936f031589`** ✓

---

## Flowchart of the Algorithm

![SHA-256 Algorithm Flowchart](imgs/SHA256_flowchart.png)

The security comes from the combination of:
- **Diffusion** — every output bit depends on every input bit (via 64 rounds of mixing)
- **Non-linearity** — Ch and Maj make the relationship between input and output non-invertible
- **Avalanche effect** — a 1-bit change in input completely changes the output
