---
layout: default
title: kHeavyHash: A Technical Overview @todo
nav_order: 2
---

# A Technical Overview of the kHeavyHash Proof-of-Work Algorithm

```{note}
This is a work in progress.  This note will be removed after a proofreading.  Feel free to contribute by asking questions where further clarification is required, or by providing constructive feedback.
```

## Abstract

This paper describes kHeavyHash, a Proof-of-Work (PoW) algorithm designed for cryptocurrency mining. kHeavyHash leverages a combination of standard cryptographic primitives, including the customizable SHAKE (cSHAKE) hash function from the SHA-3 family, and a modern pseudorandom number generator (PRNG) from the xoshiro family, specifically xoshiro256++. A core distinguishing feature of kHeavyHash is its reliance on a dynamically generated, high-dimensional, and full-rank matrix, which aims to introduce characteristics potentially beneficial for decentralized mining or specific hardware considerations.

## 1. Introduction

Proof-of-Work algorithms are fundamental to the security and consensus mechanisms of many cryptocurrencies. They require participants (miners) to expend computational effort to solve a difficult, but verifiable, mathematical problem, thereby adding new blocks of transactions to a blockchain. kHeavyHash introduces a unique approach by incorporating complex matrix operations and relying on a dynamically generated, full-rank matrix within its core hashing process. This design choice implies an intention to influence mining hardware efficiency or ensure specific statistical properties of the PoW solution.

## 2. Algorithm Overview

The kHeavyHash algorithm takes a header hash, timestamp, and nonce as inputs. It proceeds through several distinct phases:

1. Initial Header Hashing: The input components are assembled into a header and subjected to a cryptographic hash function.

2. Matrix Initialization: A large, square matrix is generated using a pseudorandom number generator, with a crucial step to ensure the matrix achieves full rank.

3. Vector Initialization: A compact vector is derived from the result of the initial header hashing.

4. Matrix-Vector Product: A core computational step involves multiplying the generated matrix by the initialized vector.

5. Final Digest Calculation: The result of the matrix-vector product is combined with parts of the initial header hash output, followed by a final cryptographic hash and byte reversal to produce the ultimate Proof-of-Work hash.

## 3. Detailed Algorithm Description

Inputs:
• Header Hash: A 32-byte (256-bit) input representing a preliminary block header hash
• Timestamp: An 8-byte (64-bit) integer representing the block's timestamp
• Nonce: An 8-byte (64-bit) unsigned integer, typically iterated by miners to find a valid hash

Process:
### 3.1 Step 1: Header Construction and Initial Hashing
The algorithm begins by constructing a byte array header by concatenating the initial 32-byte Header Hash, the 8-byte Timestamp, 32 bytes of zeros, and the 8-byte Nonce
. This combined header is then cryptographically hashed using cSHAKE256 with a specific customization string, "ProofOfWorkHash", and a requested output length of 32 bytes
.

Pseudocode for Header Construction and Initial Hashing:
```
function INITIAL_HASHING(header_hash, timestamp, nonce):
  header_bytes = header_hash || INT_TO_BYTES(timestamp, 8) || ZERO_BYTES(32) || INT_TO_BYTES(nonce, 8)
  hashed_header = cSHAKE256(header_bytes, 32, "ProofOfWorkHash", "")
  return hashed_header
end function
```

### 3.2 Step 2: Matrix Initialization
A size x size matrix (where size is a constant, set to 64
) is generated and populated with 16-bit unsigned integer values
. This matrix generation process is critical:
• PRNG Initialization: A Xoshiro256PlusPlusHasher is initialized using four 64-bit unsigned integers derived from the first 32-byte input Header Hash
. Xoshiro256++ is a modern, high-quality pseudorandom number generator known for its speed and statistical properties, passing strong statistical tests [1, 9, Xorshift - Wikipedia]. It is a 64-bit generator with a 256-bit state [1, Xorshift - Wikipedia].
• Iterative Matrix Population and Rank Check: The matrix is populated iteratively. In each iteration, the Xoshiro256PlusPlusHasher generates 64-bit pseudorandom values
. These values are used to fill sections of the matrix. Crucially, after each full population of the matrix, its rank is calculated. The process continues to generate and fill the matrix until it achieves a full rank of size (i.e., 64)
.
    ◦ The calculateRank function, which determines the rank, involves converting the matrix elements to floating-point numbers and performing a process akin to Gaussian elimination
. This ensures the generated matrix is non-singular and has maximal linear independence among its rows/columns. The use of epsilon (1e-9) accounts for floating-point precision issues
. The full-rank requirement likely contributes to better diffusion and statistical properties in the subsequent matrix multiplication.

Pseudocode for Matrix Initialization:
```
function INITIALIZE_MATRIX(s0, s1, s2, s3):
  hasher = NewXoshiro256PlusPlusHasher(s0, s1, s2, s3)
  mat = new MATRIX(size, size)
  while CALCULATE_RANK(mat) != size:
    for i from 0 to size-1:
      for j from 0 to size-1 step iterations: // iterations = size / 4 = 16 [3]
        value = hasher.Next() // 64-bit value
        for k from 0 to iterations-1:
          mat[i][j+k] = (value >> (4 * k)) & 0x0f // Extract 4-bit chunks
  return mat
end function
```

### 3.3 Step 3: Vector Initialization
A vector v of size (64) 16-bit unsigned integers is created. This vector is populated by taking 4-bit chunks from the 32-byte hashed_header (the output from Step 1)
. Each byte from hashed_header contributes two 4-bit values to the v vector (one from the upper 4 bits, one from the lower 4 bits)
.

Pseudocode for Vector Initialization:
```
function INITIALIZE_VECTOR(hashed_header_bytes):
  v = new VECTOR(size)
  for i from 0 to (size/2)-1:
    v[i*2]   = (hashed_header_bytes[i] >> 4) & 0x0f
    v[i*2+1] = hashed_header_bytes[i] & 0x0f
  return v
end function
```

### 3.4 Step 4: Matrix-Vector Product Calculation
A product vector p of size (64) 16-bit unsigned integers is computed by multiplying the mat (from Step 2) by the v vector (from Step 3)
. This involves a standard matrix-vector multiplication, where each element p[i] is the sum of the products of mat[i][j] and v[j] across all j. After summing, each p[i] value is right-shifted by 10 bits (>> 10)
, effectively scaling down the result.

Pseudocode for Matrix-Vector Product:
```
function MATRIX_VECTOR_PRODUCT(mat, v):
  p = new VECTOR(size)
  for i from 0 to size-1:
    s = 0
    for j from 0 to size-1:
      s = s + (mat[i][j] * v[j])
    p[i] = s >> 10
  return p
end function
```

### 3.5 Step 5: Digest Calculation and Final Hashing
A final 32-byte digest is computed. For each byte of the digest, it is calculated by XORing the corresponding byte from the hashed_header (output of Step 1) with two 4-bit chunks from the p vector (output of Step 4)
. Specifically, p[i*2] provides the upper 4 bits and p[i*2+1] provides the lower 4 bits for digest[i]
.
This digest is then subjected to a second cSHAKE256 hash, using the customization string "HeavyHash", and requesting a 32-byte output
. Finally, the bytes of this second hash output are reversed, yielding the ultimate kHeavyHash Proof-of-Work hash
.

Pseudocode for Final Digest and Hashing:
```
function FINAL_HASHING(hashed_header_bytes, p_vector):
  digest = new BYTE_ARRAY(32)
  for i from 0 to 31: // Corresponds to 32 bytes
    digest[i] = hashed_header_bytes[i] XOR ( (p_vector[i*2] << 4) OR p_vector[i*2+1] )

  final_hash = cSHAKE256(digest, 32, "HeavyHash", "")
  // Reverse bytes of final_hash
  reversed_final_hash = REVERSE_BYTES(final_hash)
  return reversed_final_hash
end function
```

## 4. Underlying Cryptographic Primitives

### 4.1 cSHAKE256
cSHAKE (customizable SHAKE) is a function derived from the SHA-3 Standard, specifically from the KECCAK cryptographic primitive [187, 199, 201, SHA-3 - Wikipedia]. It is an eXtendable-Output Function (XOF), meaning it can produce outputs of any desired bit length
. cSHAKE is designed to provide domain separation
, allowing users to customize its behavior through two optional parameters:
• Function-Name String (N): Used by NIST to define official SHA-3 derived functions (e.g., KMAC, TupleHash, ParallelHash)
.
• Customization String (S): Provided by the user to define a variant of the function for specific applications. Using different customization strings for cSHAKE makes it "extremely unlikely" that the computations will yield the same result, thus preventing collisions between unrelated applications
.
cSHAKE256, specifically, provides a 256-bit security strength against generic attacks
. In kHeavyHash, cSHAKE256 is used with two distinct customization strings ("ProofOfWorkHash" and "HeavyHash") to ensure unique functional behavior at different stages of the algorithm

### 4.2 Xoshiro256++
Xoshiro (xor/shift/rotate) is a family of pseudorandom number generators (PRNGs) introduced by David Blackman and Sebastiano Vigna [17, Xorshift - Wikipedia]. These generators are favored for their high speed and excellent statistical quality, often outperforming older generators like Mersenne Twister in statistical tests [1, 9, 16, Xorshift - Wikipedia].
Xoshiro256++ is a specific variant within this family. It is a 64-bit generator with a 256-bit internal state [9, Xorshift - Wikipedia]. It uses a combination of XOR, shift, and rotate operations to advance its internal state and produce output [1, Xorshift - Wikipedia]. The "++" (plusplus) scrambler, applied to two state words, involves summing them, rotating the sum, and then adding the first word again [33, Xorshift - Wikipedia]. This design helps eliminate bits of low linear complexity, making it a "strong scrambler"
. It is noted for passing all statistical tests known to its authors and being 3-dimensionally equidistributed. In kHeavyHash, Xoshiro256++ is crucial for deterministically generating the complex, full-rank matrix required for the PoW calculation

## 5. Differences between kHeavyHash and HeavyHash

kHeavyHash distinguishes itself from its conceptual predecessor, "HeavyHash" (or "Heavy Hash"), primarily in its choice and application of the underlying cryptographic hash function. While "HeavyHash" employs the generic SHA-256, kHeavyHash leverages cSHAKE256, a customizable eXtendable-Output Function (XOF) that is part of the SHA-3 derived functions. This shift introduces significant differences, especially concerning customization and explicit domain separation.

### 5.1 HeavyHash" and SHA-256
In a traditional "HeavyHash" scheme, the cryptographic puzzle's core involves evaluating a hash function, which can be instantiated by an existing hash function like SHA-256. SHA-256 is a fixed-length hash function, meaning its output is always 256 bits (32 bytes). It is a member of the SHA-2 family of algorithms. While SHA-256 provides strong cryptographic security properties against collision and preimage attacks, its design primarily uses an implicit domain separation mechanism by appending a suffix (e.g., `01` for SHA-3 instances) to the message internally to prevent collisions between different applications of the Keccak hash function. However, it does not offer the explicit, user-definable domain separation capabilities found in cSHAKE256.

### 5.2 kHeavyHash and cSHAKE256
kHeavyHash explicitly integrates **cSHAKE256** into its core processing, specifically at two distinct stages:
1.  **Header Generation**: The `header` array, incorporating the initial hash, timestamp, and nonce, is processed by `CShake256` with the function-name `N = "ProofOfWorkHash"` and a fixed output length of 32 bytes.
2.  **Final Digest Calculation**: The `digest` of the matrix product (combined with the header) is then subjected to a final hashing step using `CShake256` with the function-name `N = "HeavyHash"` and a fixed output length of 32 bytes.

cSHAKE256 is defined by NIST as a customizable variant of the SHAKE function. SHAKE, in turn, is an eXtendable-Output Function (XOF), meaning it can produce output of any desired bit length, unlike fixed-length hash functions. This is achieved by defining cSHAKE in terms of KECCAK[c], the underlying sponge function on which the SHA-3 standard is based.

The key technical distinctions and advantages of kHeavyHash's approach with cSHAKE256 are:

*   **Explicit Customization and Domain Separation**:
    *   cSHAKE256 allows for two additional parameters: a **function-name bit string (N)** and a **customization bit string (S)**.
    *   In kHeavyHash, the distinct `N` values ("ProofOfWorkHash" and "HeavyHash") are used for different stages of the computation. This **explicitly leverages the domain separation feature** of cSHAKE.
    *   This design ensures that two cSHAKE computations with different `N` or `S` strings produce **unrelated outputs**. For example, `cSHAKE(X, L, "ProofOfWorkHash", S)` and `cSHAKE(X, L, "HeavyHash", S)` are treated as completely different functions, even with identical inputs otherwise. This property is crucial for preventing related-key attacks or unintended collisions across different cryptographic contexts.
    *   In contrast, SHA-256 provides only a standard, implicit domain separation, lacking the flexibility for user-defined or function-specific context separation within the algorithm itself.

*   **Underlying Cryptographic Primitive**:
    *   SHA-256 belongs to the SHA-2 family, based on a Merkle–Damgård construction, which is different from SHA-3's underlying Keccak sponge construction.
    *   cSHAKE256 is built directly upon the **KECCAK sponge function**. The sponge construction allows data to be "absorbed" into a state and then "squeezed" out, preventing length extension attacks that Merkle–Damgård constructions (like SHA-2) are susceptible to.

*   **Output Flexibility (XOF Nature)**:
    *   While kHeavyHash uses cSHAKE256 to produce a fixed 32-byte output for its internal steps, cSHAKE256 itself is an XOF, capable of **variable-length outputs of any bit length**. In contrast, SHA-256 is strictly a fixed-output hash function.
    *   It is important to note that cSHAKE, unlike other SHA-3 derived functions like KMAC, TupleHash, and ParallelHash, retains the XOF property that a shorter output is a prefix of a longer output when `N` and `S` are the same.

*   **Security Strength and Integrity**:
    *   Both cSHAKE256 and SHA-256 are designed to provide a 256-bit security strength against generic attacks.
    *   The explicit domain separation in cSHAKE256 **enhances the robustness** of cryptographic applications by making it extremely unlikely that two computations with different customization strings will yield the same answer. This design choice is analogous to "strong typing" in programming languages for cryptographic functions.
    *   NIST explicitly states that for any legal `N` and `S`, cSHAKE offers the **exact same security properties as SHAKE**, meaning there are no "weak" values for `N` or `S`.

*   **Implementation Considerations**:
    *   Implementations of cSHAKE can **precompute** the processing of the function name (`N`) and customization string (`S`), which can improve performance when reusing the same choices of `N` and `S` in multiple executions. This is a potential efficiency benefit for kHeavyHash due to its fixed `N` strings.

In summary, the transition from generic SHA-256 in "HeavyHash" to cSHAKE256 in kHeavyHash represents a move towards a more **modern, flexible, and robust cryptographic primitive**. The explicit domain separation provided by cSHAKE256 with its `N` and `S` parameters adds a layer of security and clarity, ensuring that distinct applications or stages within the kHeavyHash algorithm are cryptographically isolated, mitigating potential collision risks and enhancing overall cryptographic hygiene.

Imagine a locksmith who uses a single, generic key-cutting machine (SHA-256) for all his work. He might mentally label the keys for different purposes, but the machine itself doesn't differentiate. Now, imagine a more advanced locksmith with a machine that allows him to embed specific, unalterable tags directly into each key's design (cSHAKE256's N and S parameters), indicating whether it's a "house key" or a "car key" or a "safe deposit box key." Even if two keys happen to have the same "cut" for their main purpose, the embedded "tags" make them fundamentally different, preventing one from accidentally or maliciously being used for the other. This explicit tagging adds a layer of safety and specificity that the generic machine lacks.

## 6. Conclusion
kHeavyHash is a PoW algorithm that integrates advanced cryptographic primitives and linear algebraic concepts to generate its solution. Its reliance on cSHAKE256 provides robust cryptographic hashing and domain separation, while the use of Xoshiro256++ for matrix generation ensures high-quality pseudorandomness. The requirement for a dynamically generated, full-rank matrix is a notable feature, setting it apart from simpler hash-based PoW algorithms and potentially influencing the efficiency and type of hardware best suited for its computation.
