---
layout: default
title: kHeavyHash: A Technical Overview
nav_order: 1
---

# kHeavyHash: A Technical Overview

```{note}
This is a work in progress.  This note will be removed after a proofreading.  Feel free to contribute by asking questions where further clarification is required, or by providing constructive feedback.
```

## Abstract

kHeavyHash is a proof-of-work (PoW) algorithm that combines the extendable-output cryptographic hashing function `cSHAKE256` from the SHA-3 family with a high-performance, deterministic pseudorandom number generator (PRNG) known as `xoshiro256++`. The algorithm is designed for computational complexity and memory hardness, utilizing a dynamically generated full-rank 64x64 matrix of 16-bit integers as a core element. This document elaborates on the architecture and mechanics of kHeavyHash, detailing the operations from preprocessing through final digest, and including a review of the cryptographic primitives and how they are used securely within the protocol.

## 1. Introduction

Proof-of-Work (PoW) algorithms provide decentralized consensus and security in blockchain networks by requiring miners to solve resource-intensive problems. Unlike simple hashing-based PoW schemes, kHeavyHash introduces structured mathematical computation, primarily in the form of linear algebra. By incorporating pseudorandom full-rank matrix generation and matrix-vector multiplications, kHeavyHash aims to increase the computational cost for specialized hardware while retaining verifiability.

This technical overview discusses the components and workflow of kHeavyHash and explains how each part contributes.

## 2. Algorithm Overview

**Inputs:**
- Header Hash (32 bytes)
- Timestamp (8 bytes)
- Padding (32 bytes of zeros)
- Nonce (8 bytes)

**Key Steps:**
1. Header construction
2. Initial hashing with cSHAKE256
3. Matrix initialization with xoshiro256++
4. Vector initialization
5. Matrix-vector multiplication
6. Final digest construction and hashing with cSHAKE256

## 3. Detailed Algorithm Description

### Step 1: Header Construction
The header is composed of four parts concatenated into an 80-byte sequence:
- 32 bytes: Header hash (precomputed hash of the block header)
- 8 bytes: Timestamp (current UNIX time in 64-bit little-endian format)
- 32 bytes: Zero padding
- 8 bytes: Nonce (changing value to affect output)

### Step 2: Initial Hashing
Using cSHAKE256 with the customization string `"ProofOfWorkHash"`, the header is hashed to a 32-byte digest. cSHAKE256 provides strong domain separation by customizing the hash function for a specific purpose.

### Step 3: Matrix Initialization
A 64x64 matrix of 16-bit integers is generated. This is accomplished using xoshiro256++ seeded with the initial 32-byte digest. The PRNG produces deterministic but high-quality pseudorandom values that populate the matrix. To ensure invertibility and spread of values, the matrix must be full-rank (all rows linearly independent), and generation repeats until this condition is met.

### Step 4: Vector Initialization
The initial digest is interpreted as 64 4-bit values, producing a vector of shape (64,). This vector is prepared for use in the matrix-vector multiplication.

### Step 5: Matrix-Vector Multiplication
The 64x64 matrix is multiplied by the 64-entry vector using standard integer arithmetic. The results are normalized (typically using right-shifting or modular operations) to fit within a reduced range suitable for hashing.

### Step 6: Final Digest and Output Hashing
The result vector is hashed again using cSHAKE256 with the customization string `"HeavyHash"`. The resulting digest is reversed byte-wise to finalize the hash output.

## 4. Cryptographic Primitives

### cSHAKE256 (from SHA-3 / FIPS 202 / SP800-185)

cSHAKE256 is an extendable-output function (XOF) derived from SHAKE256, itself built upon the KECCAK sponge construction. It accepts a customization string (S) and function name (N), allowing unique instances of the hash for domain separation.

**A typical cSHAKE256 call:**

```
cSHAKE256(X, L, N, S)
```

Where:
- `X` is the input bit string
- `L` is the output length in bits
- `N` is the function name (often empty)
- `S` is a customization string (e.g., "ProofOfWorkHash")

Internally, cSHAKE applies a preprocessing `bytepad()` on encoded customization values and appends the actual input. The KECCAK-f permutation is applied to the resulting padded string.

This domain separation ensures that outputs for different uses (even with the same input) are unrelated, bolstering resistance to preimage and collision attacks.

### xoshiro256++ PRNG

xoshiro256++ is a high-speed, high-quality PRNG designed for statistical robustness and computational efficiency. It consists of:
- A 256-bit internal state (four 64-bit unsigned integers)
- A scrambling function that sums, rotates, and sums again

**Simplified pseudocode:**

```c
uint64_t result = rotl(s[0] + s[3], 23) + s[0];
s[2] ^= s[0];
s[3] ^= s[1];
s[1] ^= s[2];
s[0] ^= s[3];
s[2] ^= t;
s[3] = rotl(s[3], 45);
```

This pattern ensures excellent distribution and performance, and it's efficient enough for SIMD and hardware acceleration.

## 5. Diagram Reference

![kHeavyHash Diagram](assets/img/kHeavyHash.png)

## Appendix A: Details on xoshiro256++ PRNG

xoshiro256++ is part of a family of scrambled linear generators that use nonlinear operations (like rotation and addition) to improve statistical performance and hide regularities.

### Features:
- 256-bit internal state ensures period of 2^256
- Low latency and high throughput (0.86 ns per 64-bit word)
- Passes rigorous randomness tests (e.g. BigCrush, PractRand)
- Lightweight logic suitable for FPGAs and ASICs

## Appendix B: Details on cSHAKE256

cSHAKE256 is defined in NIST SP 800-185 as:

```
cSHAKE256(X, L, N, S) = KECCAK[512](bytepad(encode(N) || encode(S), 136) || X || 00, L)
```

- Bitrate: 136 bytes (1088 bits)
- Capacity: 64 bytes (512 bits)

It retains the indifferentiability properties of KECCAK while enabling use-case specific customization. This makes it highly adaptable for secure hashing in constrained environments.

## Appendix C: Security and Implementation Considerations

- `cSHAKE256` provides security backed by NIST standards.
- `xoshiro256++` introduces structured randomness in a reproducible way, ideal for mining protocols.
- Full-rank matrix generation significantly raises the cost of parallelized or ASIC-optimized hashing.
- The algorithm is hardware-friendly: primarily composed of integer arithmetic, XOR, and small-width memory access.
