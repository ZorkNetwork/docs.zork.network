---
layout: default
title: MimbleWimble Extension Block
nav_order: 3
published: false
nav_exclude: true
search_exclude: true
hide: true
---

# MimbleWimble Extension Block

This document provides a comprehensive technical overview of the **MimbleWimble Extension Block (MWEB)** feature as implemented in **Zorkcoin** (via its parent fork, Litecoin). It is designed to serve as a singular reference for users, developers, and blockchain experts.

---

## 1. High-Level Overview (User Perspective)

### 1.1 What is MWEB?
MWEB, short for **MimbleWimble Extension Block**, is a privacy-focused protocol integrated into the Zorkcoin blockchain. The name reflects the design goal of limiting what observers can infer about transaction amounts and participants on the public ledger. 

Zorkcoin implements this through **Extension Blocks (EB)**, which operate as a parallel sidechain to the main Zorkcoin network. This architecture allows users to "opt-in" to privacy features without disrupting the transparency and security of the primary chain.

### 1.2 Core Benefits
*   **Confidential Transactions**: Hides the exact amount of Zorkcoin transferred.
*   **Enhanced Fungibility**: Ensures all coins are equal regardless of their transaction history, mimicking the privacy of physical cash.
*   **Scalability**: Utilizes "cut-through" technology to prune spent transaction data, reducing the long-term storage requirements for nodes.
*   **Address Privacy**: Obscures sender and recipient identities using specialized stealth addresses.

### 1.3 The "Peg-In" and "Peg-Out" Model
To use MWEB, users perform two primary actions:
1.  **Peg-In**: Moving funds from the standard Zorkcoin chain into the MWEB extension block.
2.  **Peg-Out**: Moving funds back from MWEB to the regular, transparent Zorkcoin network.

---

## 2. System Architecture (Implementation Level)

### 2.1 Extension Blocks (LIP-0002)
Instead of modifying Zorkcoin’s base layer, the protocol uses the **Extension Block framework (LIP-0002)**. This allows for a "two-way peg" where UTXOs are locked on one side to be released on the other, maintaining a strict 1-1 value mapping.

### 2.2 The Hogwarts Express (HogEx)
The connection between the main chain and the extension block is managed by a specialized transaction called the **HogEx (Hogwarts Express)**.
*   The HogEx is the final transaction in a block.
*   It spends coins from the previous block’s **HogAddr** (Hogwarts Address) and directs them to the current block's HogAddr, which holds the total supply currently "pegged into" the MWEB environment.
*   **HogAddr** is an "anyone-can-spend" Bech32 witness program (version 8) that commits to the MWEB block hash.

### 2.3 Synchronization
MWEB is a **synchronized sidechain**. Every Zorkcoin block contains exactly one corresponding MWEB block (if MWEB is active). This prevents fork-related issues between the two chains and ensures consensus consistency.

---

## 3. Cryptographic Foundations (Technical Deep Dive)

### 3.1 Pedersen Commitments
MWEB replaces explicit transaction amounts with **Pedersen Commitments**. A commitment $C$ is defined as:
$$C = vH + rG$$
Where:
*   **$v$** is the value of the transaction.
*   **$r$** is the **blinding factor** (a secret random value).
*   **$H$** and **$G$** are generator points on the secp256k1 elliptic curve.

Because the blinding factor $r$ is known only to the transaction participants, an outside observer cannot determine the value $v$. However, the network can still verify that no coins were created out of thin air by confirming that the sum of output commitments minus the sum of input commitments equals zero.

### 3.2 Bulletproofs (Rangeproofs)
To prevent users from creating "negative" amounts (which would effectively mint new coins), every output includes a **Bulletproof**. This is a zero-knowledge rangeproof that mathematically proves the committed value is within a valid range $[0, 2^{64})$ without revealing the value itself.

### 3.3 Transaction Kernels
Every MWEB transaction produces a **Kernel**, which acts as proof of the transaction’s validity.
*   **Excess Commitment ($E$)**: The part of the commitment where values cancel out, leaving only a public key derived from the blinding factors.
*   **Kernel Signature**: A Schnorr signature proving the creator knows the blinding factors.
*   **Feature Bits**: Specifies kernel properties such as fees, peg-ins, peg-outs, and height locks.

---

## 4. One-Sided Transactions and DKSAP (Expert Level)

### 4.1 LIP-0004: Non-Interactive Transactions
A major advancement in Zorkcoin’s MWEB implementation (via LIP-0004) is the support for **one-sided transactions**. Unlike original MimbleWimble, which required sender and receiver to communicate off-chain to build a transaction, Zorkcoin uses the **Dual-Key Stealth Address Protocol (DKSAP)**.

### 4.2 Stealth Address Structure
An MWEB stealth address consists of two secp256k1 public keys:
1.  **Scan Key ($A_i$)**: Allows the wallet to "scan" the blockchain to identify outputs belonging to the user.
2.  **Spend Key ($B_i$)**: Required to spend the funds.

### 4.3 Output Identification
To receive funds, a receiver provides a stealth address. The sender uses an ephemeral key and Diffie-Hellman exchange to derive a one-time public key for that specific output. The receiver’s wallet identifies these outputs by checking for a **view tag**, which avoids the need for heavy ECC calculations for every output on the chain.

---

## 5. Comparative Analysis: Zorkcoin/Litecoin vs. Others

| Feature | Zorkcoin / Litecoin (MWEB) | Grin | Beam | MimbleWimbleCoin (MWC) |
| :--- | :--- | :--- | :--- | :--- |
| **Integration** | Opt-in Extension Block | Pure MimbleWimble Chain | Pure MimbleWimble Chain | Dedicated MW Chain |
| **Interaction** | Non-Interactive (One-sided) | Interactive (Manual) | BBS-based / Interactive | Interactive / Non-interactive |
| **Address Type** | Stealth Addresses (DKSAP) | No Addresses | Publicly shared IDs (off-chain) | No Addresses (Standard MW) |
| **Monetary Policy** | Scarcity (LTC-based) | Linear Emission (Infinite) | Deflationary (Hard Cap) | Hard Cap |
| **Governance** | Open Source / Foundation | Decentralized / Donations | VC-backed / Startup | Community-led |
| **Verification** | Batch Verification | Batch Verification | Batch Verification | Batch Verification |

### 5.1 Key Differentiators
*   **Accessibility**: Zorkcoin’s MWEB is uniquely suited for users who value the stability of a 2011-era codebase but desire modern privacy options.
*   **Convenience**: By implementing LIP-0004 (one-sided transactions), Zorkcoin resolves the "online requirement" hurdle that hindered Grin and early Beam implementations.
*   **Risk Mitigation**: As an extension block, MWEB can be updated or isolated via soft-fork without compromising the underlying Zorkcoin L1.

---
*Note: This technical documentation is based on current MWEB specifications as of 2024-2025. Developers should consult LIP-0002, LIP-0003, and LIP-0004 for exact implementation standards.*