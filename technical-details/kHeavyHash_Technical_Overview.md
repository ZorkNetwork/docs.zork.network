---
layout: default
title: kHeavyHash Technical Overview
nav_order: 1
---

# kHeavyHash Technical Specification

kHeavyHash is a proof‑of‑work algorithm that combines the extendable‑output function **cSHAKE256** (SHA‑3 family) with the high‑performance pseudorandom generator **xoshiro256++**. Its distinguishing feature is a dynamically generated, full‑rank **64×64 matrix of 16‑bit values** used in a matrix–vector multiplication step. This specification is precise and unambiguous, matches the reference implementation, and is suitable for interoperable implementations and normative verification.

---

## Introduction
kHeavyHash augments a conventional hash‑based PoW with a heavy linear‑algebra step to increase implementation cost for specialized hardware while keeping verification fast. This document fixes all determinism, endianness, and algebraic semantics required for portable implementations and verification, and aligns with the reference implementation behavior.

---

## Notation and Inputs
- **Byte order:** **little‑endian** for multi‑byte integer encodings unless explicitly stated.  
- **Types:** `u8` = 8‑bit unsigned, `u16` = 16‑bit unsigned, `u32` = 32‑bit unsigned, `u64` = 64‑bit unsigned.  
- **Concatenation:** `||` denotes byte concatenation.  
- **Nibble ordering:**  
  - **Low‑nibble‑first**: `(word >> (4*k)) & 0xF` with `k=0` the least significant nibble.  
  - **High‑nibble‑first**: `(byte >> 4) & 0xF` then `byte & 0xF`.  
- **Inputs:**  
  - **Header hash:** 32 bytes (opaque).  
  - **Timestamp:** 8 bytes, little‑endian `u64`.  
  - **Nonce:** 8 bytes, little‑endian `u64` (miner‑controlled).

---

## Algorithm Specification

### Overview
1. Build an 80‑byte work order and compute **PrePowHash** = `cSHAKE256(work_order, L=256, N="", S="ProofOfWorkHash")`.  
2. Seed **xoshiro256++** with the first 32 bytes of the work order (the PrePowHash input block) interpreted as four little‑endian `u64` words.  
3. Generate a 64×64 matrix of `u16` entries from the PRNG stream; require **full rank** under 16‑bit unsigned wraparound semantics. If not full rank, refill using the next PRNG words until full rank.  
4. Derive a 64‑element `u16` vector from `PrePowHash` (nibbles high‑first).  
5. Compute `p = M × v` with exact integer accumulation, right‑shift each accumulator by 10 bits, pack low 4 bits of `p` elements into bytes, XOR with `PrePowHash`, and compute final `cSHAKE256` with customization `"HeavyHash"`.  
6. Reverse the final 32‑byte output to produce the PoW hash (little‑endian comparison order).

### Work Order and Initial Hash
- **Work order (80 bytes):**
  ```
  work_order = header_hash (32 bytes)
             || timestamp_le64 (8 bytes)
             || zeros(32)      (32 bytes)
             || nonce_le64     (8 bytes)
  ```
- **PrePowHash:**
  ```
  PrePowHash = cSHAKE256(work_order, L=256, N="", S="ProofOfWorkHash")
  ```
  - `PrePowHash` is 32 bytes.

### PRNG Seeding and Stream
- **Seed source:** the reference implementation seeds xoshiro256++ from the **first 32 bytes of the work order** (the PrePowHash input block), interpreted as four little‑endian `u64` words:
  ```
  seed_word[k] = little_endian_u64(work_order[8*k .. 8*k+7])  for k = 0..3
  ```
- **PRNG:** canonical xoshiro256++ (256‑bit state, 64‑bit outputs). Use it only as a deterministic PRNG for matrix generation.

### Matrix Generation (64×64 `u16`)
- **Layout:** `matrix[row][col]` with `row = 0..63`, `col = 0..63`. Row‑major storage. Size = **8192 bytes**.  
- **Mapping PRNG → entries (deterministic):**
  - For each 64‑bit PRNG word `r`, extract 16 nibbles **low‑nibble‑first**:
    ```
    for k in 0..15:
        nibble[k] = (r >> (4*k)) & 0xF
    ```
  - Each nibble becomes a `u16` matrix entry (upper 12 bits zero). Fill matrix row‑major until all 4096 entries are set.  
- **Full‑rank requirement:** compute rank under **16‑bit unsigned wraparound semantics** (see Algebraic Semantics). If `rank < 64`, continue the PRNG stream and **refill the entire matrix** using the next PRNG words until a full‑rank matrix is produced. Expected iterations are typically 1–2.

### Algebraic Semantics (Normative)
- **Normative semantics:** interpret matrix and vector entries as elements of the ring \(\mathbb{Z}/2^{16}\mathbb{Z}\) (standard `u16` wraparound arithmetic).  
- **Rank computation:** perform exact Gaussian elimination using modular integer arithmetic over \(\mathbb{Z}/2^{16}\mathbb{Z}\). Treat an element as invertible only if it is a unit modulo \(2^{16}\) (i.e., odd).  
- **Implementation note:** the reference implementation uses a floating‑point elimination with `eps = 1e‑9` as a pragmatic full‑rank check; this is **non‑normative**. Verifiers must use exact integer modular elimination for normative verification.

### Vector Initialization
- **Derive `v[0..63]` from `PrePowHash` (high‑nibble‑first):**
  ```
  for i in 0..31:
      v[i*2]   = (PrePowHash[i] >> 4) & 0xF
      v[i*2+1] = PrePowHash[i] & 0xF
  ```
- Each `v[k]` is a `u16` in 0..15.

### Matrix–Vector Multiplication
- **Computation:**
  ```
  for i in 0..63:
      acc = 0u64
      for j in 0..63:
          acc += (u32)matrix[i][j] * (u32)v[j]   // 16×16 → 32, accumulate in 64
      p[i] = (u32)(acc >> 10)
  ```
- **Types:** use `u64` accumulation to avoid overflow. The right shift by 10 reduces magnitude.

### Final Packing and Output
- **Pack low 4 bits of `p` into bytes and XOR with PrePowHash:**
  ```
  for i in 0..31:
      packed = ((p[i*2] & 0xF) << 4) | (p[i*2+1] & 0xF)
      digest[i] = PrePowHash[i] XOR (u8)packed
  ```
- **Final hash and output:**
  ```
  final = cSHAKE256(digest, L=256, N="", S="HeavyHash")
  output = reverse_bytes(final)   // output[i] = final[31 - i]
  ```
- **PoW hash:** `output` is the 32‑byte PoW result used for target comparison.

---

## Reference Pseudocode
```pseudo
function kHeavyHash(header_hash[32], timestamp_le64, nonce_le64):
    work_order = header_hash || timestamp_le64 || zeros(32) || nonce_le64
    PrePowHash = cSHAKE256(work_order, L=256, N="", S="ProofOfWorkHash")  // 32 bytes

    // Seed xoshiro from first 32 bytes of work_order (little-endian u64 words)
    for k in 0..3:
        seed_word[k] = le64(work_order[8*k .. 8*k+7])
    xoshiro.seed(seed_word[0..3])

    // Generate matrix until full rank (u16 wraparound semantics)
    loop:
        matrix = new u16[64][64]
        idx = 0
        while idx < 4096:
            r = xoshiro.next_u64()
            for k in 0..15:
                nib = (r >> (4*k)) & 0xF          // low-nibble-first
                matrix[idx / 64][idx % 64] = (u16)nib
                idx += 1
        if is_full_rank_mod_2pow16(matrix):
            break
        // else continue loop (use next PRNG words to refill)

    // Vector v from PrePowHash (high-nibble-first)
    for i in 0..31:
        v[i*2]   = (PrePowHash[i] >> 4) & 0xF
        v[i*2+1] = PrePowHash[i] & 0xF

    // Multiply
    for i in 0..63:
        acc = 0u64
        for j in 0..63:
            acc += (u32)matrix[i][j] * (u32)v[j]
        p[i] = (u32)(acc >> 10)

    // Pack and XOR
    for i in 0..31:
        packed = ((p[i*2] & 0xF) << 4) | (p[i*2+1] & 0xF)
        digest[i] = PrePowHash[i] XOR (u8)packed

    final = cSHAKE256(digest, L=256, N="", S="HeavyHash")
    output = reverse_bytes(final)
    return output
```

---

## Test Vectors and Interoperability Requirements
Implementations **must** publish test vectors derived from the reference implementation. Each test vector should include:
- **Input triple:** `header_hash` (32 bytes), `timestamp` (u64), `nonce` (u64).  
- **PrePowHash** (32 bytes).  
- **PRNG seed words** (4 × `u64` little‑endian).  
- **Matrix checksum** (e.g., SHA‑256 of the 8192‑byte matrix).  
- **Sample `p` values** (e.g., first 8 `p[i]` values).  
- **Final output** (32 bytes).  

Providing these vectors removes remaining ambiguity and enables independent verification.

---

## Security, Implementation Notes, and Rationale
- **Algebraic semantics:** the reference implementation uses **16‑bit unsigned wraparound arithmetic** (ring \(\mathbb{Z}/2^{16}\mathbb{Z}\)). This specification adopts that as the normative arithmetic model for correctness and verification.  
- **Rank check:** the reference uses a floating‑point elimination with `eps = 1e‑9` as a pragmatic full‑rank test. This is **non‑normative**. Verifiers must use exact integer modular elimination over \(\mathbb{Z}/2^{16}\) for normative validation. Floating‑point checks may be used as fast prechecks in implementations but not for final verification.  
- **xoshiro256++ role:** xoshiro256++ is a deterministic, non‑cryptographic PRNG used only to generate matrix entries. The cryptographic strength of the scheme relies on cSHAKE256; xoshiro256++ must not be treated as a cryptographic RNG.  
- **Determinism and endianness:** all multi‑byte encodings and seed interpretations are **little‑endian**. PRNG→matrix uses **low‑nibble‑first**; digest→vector uses **high‑nibble‑first**. These mappings are required for interoperability.  
- **Memory footprint:** matrix = **8192 bytes**; total working set is small and GPU‑friendly.  
- **Verification cost:** one matrix–vector multiply plus two cSHAKE256 calls.  
- **Packing detail:** only the low 4 bits of each shifted `p[k]` are used when packing into bytes for XOR with `PrePowHash`. This preserves the design intent of mixing small portions of the product vector into the digest.

---

## Implementation Recommendations
- **Rank routine:** provide an exact modular Gaussian elimination over \(\mathbb{Z}/2^{16}\) for verification; floating‑point elimination may be used as a fast precheck only.  
- **Centralize nibble helpers** to avoid off‑by‑one errors.  
- **Publish test vectors** matching the reference repository tests.  
- **Document choices** (e.g., whether you use the reference floating‑point precheck) and include matrix checksum and sample `p` values in test artifacts.

