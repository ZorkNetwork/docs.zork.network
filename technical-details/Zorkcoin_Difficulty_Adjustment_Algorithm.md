---
layout: default
title: Zorkcoin™ Difficulty Adjustment Algorithm
nav_order: 100
---

# Zorkcoin™ Difficulty Adjustment Algorithm

**ZN-RFC 2: ASERT Difficulty Adjustment Algorithm (aserti3-2d)**
**Status: Active**
**Category: Standards Track**

---

## 1. Introduction
A Request for Comments (RFC) serves as a formal blueprint for technical implementations within the network. This document specifies the **Absolutely Scheduled Exponentially Rising Targets (ASERT)** algorithm, specifically the `aserti3-2d` variant, used to manage mining difficulty. **ASERT is a sophisticated control system designed to maintain a predictable block generation interval regardless of fluctuations in network computational power**. Unlike earlier mechanisms, ASERT utilizes an **Exponential Moving Average (EMA)** anchored to a fixed point in history to ensure long-term stability and resistance to manipulation.

---

## 2. Motivation and Benefits
The implementation of ASERT addresses critical vulnerabilities found in traditional Proof-of-Work (PoW) consensus models.
*   **Elimination of Oscillations:** Older algorithms often suffer from "feast or famine" cycles where blocks are found either too quickly or far too slowly. ASERT eliminates these periodic oscillations in difficulty and hashrate.
*   **Mitigation of Coin Hopping:** By smoothing adjustments, ASERT reduces the profitability gap between steady miners and "coin hoppers" who switch between blockchains to exploit difficulty drops.
*   **Predictable Confirmations:** The algorithm maintains average block intervals close to the ideal target, ensuring consistent transaction confirmation times for users.
*   **Drift Prevention:** Because it refers to an absolute schedule, ASERT prevents the long-term "drift" of block times away from the protocol's intended schedule.

---

## 3. Comparison with Native Bitcoin/Litecoin DAA
ASERT represents a fundamental shift in philosophy from the native difficulty adjustment algorithms used by Bitcoin and Litecoin.
*   **Calculation Basis:** Bitcoin and Litecoin use a **Simple Moving Average (SMA)** that adjusts only every 2016 blocks (approximately every two weeks). ASERT adjusts the difficulty **every single block**.
*   **Reference Points:** The native Bitcoin DAA is **relative**, looking only at the time taken to mine the previous 2016-block window. ASERT is **absolute**, calculating difficulty based on the time elapsed since a fixed **anchor block** (often the genesis block or a specific upgrade block).
*   **Responsiveness:** Native SMA algorithms are slow to react to sudden hash rate changes, leading to instability if large mining pools join or leave. ASERT reacts **exponentially and continuously**, providing a damping effect that prevents over-correction and feedback loops.

---

## 4. Difficulty and Hash Rate Calculations
The DAA acts as a control system where the "error signal" is the deviation of the actual time from the ideal schedule.

### 4.1 The Core Formula
The conceptual formula for determining the next target is:
`next_target = anchor_target * 2**((time_delta - ideal_block_time * (height_delta + 1)) / halflife)`.
*   **anchor_target:** The difficulty target of the historical anchor block.
*   **time_delta:** The actual seconds elapsed between the current block and the parent of the anchor block.
*   **ideal_block_time:** The targeted interval (e.g., 600 seconds).
*   **height_delta:** The number of blocks since the anchor block.
*   **halflife (τ):** A constant (set to 172,800 seconds or 2 days) defining how quickly the algorithm corrects deviations.

### 4.2 Fixed-Point Implementation
To avoid platform-dependent inconsistencies in floating-point math, the network utilizes **fixed-point integer arithmetic**. A **cubic polynomial approximation** is used to calculate the exponential $2^x$ term, ensuring identical results across all hardware and software implementations.

---

## 5. Impact of PoW Algorithm Changes
The ASERT DAA is designed to be **agnostic to the underlying hashing function**.
*   **Computational Independence:** Whether the network uses SHA256 (as in Bitcoin), Ethash (as in Ethereum), or a custom PoW like NexaPOW, the ASERT calculation remains the same.
*   **Time-Based Control:** The algorithm only measures **time and block height**. If a PoW change occurs (e.g., switching from SHA256 to a GPU-friendly algorithm), the DAA simply observes the resulting block production rate. 
*   **Adaptive Tuning:** If the new PoW algorithm allows miners to find blocks faster, the `time_delta` will decrease relative to the `height_delta`. ASERT will respond by **lowering the target** (increasing difficulty) until the rhythm returns to the `ideal_block_time`, regardless of the specific hash function being used.

---

## 6. Copyright and Legal
By the authority of the Zork Network Request for Comments process, this documentation is open for community use to ensure the transparency and security of the decentralized network.
