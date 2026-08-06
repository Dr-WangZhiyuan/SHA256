# SHA-256 Algorithm — Step-by-Step Explanation

This document describes the SHA-256 algorithm as implemented in `sha256.py`, following the FIPS 180-4 specification.

---

## Overview

SHA-256 takes an arbitrary-length input (text string or raw file bytes) and produces a fixed 256-bit (32-byte) hash digest. The algorithm operates on 512-bit blocks and uses 64 rounds of compression per block.

---

## Step 1: Constants Initialization

**Code reference:** `sha256.py` lines 7–30

Two sets of constants are defined before hashing begins:

### Round Constants (K)

64 constant 32-bit words, derived from the first 32 bits of the fractional parts of the **cube roots** of the first 64 prime numbers (2, 3, 5, 7, ..., 311).

```
K[0..63] = [0x428a2f98, 0x71374491, ..., 0xc67178f2]
```

### Initial Hash Values (H_INIT)

8 initial 32-bit words, derived from the first 32 bits of the fractional parts of the **square roots** of the first 8 prime numbers (2, 3, 5, 7, 11, 13, 17, 19).

```
H_INIT = [0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a,
           0x510e527f, 0x9b05688c, 0x1f83d9ab, 0x5be0cd19]
```

---

## Step 2: Pre-Processing (Padding)

**Code reference:** `sha256.py` lines 38–47

The message must be padded so its total length is a multiple of 512 bits (64 bytes).

1. **Convert input to bytes** — if the input is a text string, encode it as UTF-8. If the input is already raw bytes (e.g. file contents), use them directly.
2. **Append bit `1`** — in practice, append the byte `0x80`.
3. **Append zero bytes** until the message length ≡ 56 (mod 64). This ensures 8 bytes remain in the final block for the length field.
4. **Append the original message length** in bits as a 64-bit big-endian integer.

### Example

For input `"abc"` (3 bytes = 24 bits):
- After `0x80`: 4 bytes
- Zero padding: 52 bytes (to reach 56)
- Length field: 8 bytes (`0x0000000000000018` = 24 in decimal)
- Total: 64 bytes = one 512-bit block

---

## Step 3: Initialize Hash Values

**Code reference:** `sha256.py` line 50

```python
h0, h1, h2, h3, h4, h5, h6, h7 = H_INIT
```

These 8 working variables accumulate the hash state across all blocks.

---

## Step 4: Process Each 512-Bit Block

**Code reference:** `sha256.py` lines 53–95

The padded message is split into 64-byte (512-bit) blocks. Each block goes through the following sub-steps:

### Step 4a: Create the Message Schedule (W)

**Code reference:** `sha256.py` lines 57–64

The 64-byte block is expanded into 64 words (W[0..63]):

- **W[0..15]:** Directly parsed from the block — each 4 consecutive bytes form one 32-bit big-endian word.
- **W[16..63]:** Computed using:

```
σ0 = ROTR(W[j-15], 7) ⊕ ROTR(W[j-15], 18) ⊕ SHR(W[j-15], 3)
σ1 = ROTR(W[j-2], 17) ⊕ ROTR(W[j-2], 19)  ⊕ SHR(W[j-2], 10)
W[j] = (W[j-16] + σ0 + W[j-7] + σ1) mod 2^32
```

Where:
- `ROTR(x, n)` = circular right rotation of x by n bits
- `SHR(x, n)` = right shift of x by n bits (zero-fill)
- `⊕` = bitwise XOR

### Step 4b: Initialize Working Variables

**Code reference:** `sha256.py` line 67

```python
a, b, c, d, e, f, g, h = h0, h1, h2, h3, h4, h5, h6, h7
```

### Step 4c: Compression Function (64 Rounds)

**Code reference:** `sha256.py` lines 70–85

For each round `j` from 0 to 63:

```
Σ1 = ROTR(e, 6) ⊕ ROTR(e, 11) ⊕ ROTR(e, 25)
Ch  = (e AND f) ⊕ (NOT e AND g)
T1  = (h + Σ1 + Ch + K[j] + W[j]) mod 2^32

Σ0  = ROTR(a, 2) ⊕ ROTR(a, 13) ⊕ ROTR(a, 22)
Maj = (a AND b) ⊕ (a AND c) ⊕ (b AND c)
T2  = (Σ0 + Maj) mod 2^32
```

Then update the working variables:

```
h = g
g = f
f = e
e = (d + T1) mod 2^32
d = c
c = b
b = a
a = (T1 + T2) mod 2^32
```

**Logical functions explained:**
| Function | Formula | Purpose |
|----------|---------|---------|
| Ch (Choice) | `(e AND f) ⊕ (NOT e AND g)` | For each bit position, if `e=1` choose `f`, else choose `g` |
| Maj (Majority) | `(a AND b) ⊕ (a AND c) ⊕ (b AND c)` | For each bit position, output the majority value of a, b, c |
| Σ0 | `ROTR(a,2) ⊕ ROTR(a,13) ⊕ ROTR(a,22)` | Mixing function on `a` |
| Σ1 | `ROTR(e,6) ⊕ ROTR(e,11) ⊕ ROTR(e,25)` | Mixing function on `e` |

### Step 4d: Update Hash Values

**Code reference:** `sha256.py` lines 88–95

After all 64 rounds, add the compressed working variables back to the running hash:

```
h0 = (h0 + a) mod 2^32
h1 = (h1 + b) mod 2^32
h2 = (h2 + c) mod 2^32
h3 = (h3 + d) mod 2^32
h4 = (h4 + e) mod 2^32
h5 = (h5 + f) mod 2^32
h6 = (h6 + g) mod 2^32
h7 = (h7 + h) mod 2^32
```

This addition ensures that the compression function is not invertible — you cannot recover the input from the output.

---

## Step 5: Produce the Final Hash

**Code reference:** `sha256.py` lines 98–99

Concatenate h0 through h7 as 32-bit big-endian values to produce the 256-bit digest:

```
digest = h0 || h1 || h2 || h3 || h4 || h5 || h6 || h7
```

The result is output as a 64-character hexadecimal string.

---

## Helper Function: Right Rotate

**Code reference:** `sha256.py` lines 33–34

```python
def right_rotate(value, amount):
    return ((value >> amount) | (value << (32 - amount))) & 0xFFFFFFFF
```

Circular right rotation within a 32-bit word. Bits shifted off the right end wrap around to the left. The `& 0xFFFFFFFF` mask ensures the result stays within 32 bits.

---

## Verification

The implementation produces correct results for all standard test vectors:

| Input | SHA-256 Output |
|-------|---------------|
| `""` (empty) | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |
| `"abc"` | `ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad` |
| `"hello"` | `2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824` |

---

## References

- NIST FIPS 180-4: Secure Hash Standard (SHS)
- Implementation: `sha256.py` in this repository
