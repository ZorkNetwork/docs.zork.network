---
layout: default
title: From a Hobby Node to Solo Mining — A Gentle Map
nav_order: 6
lang: en
description: "A high-level map of solo mining on Zork Network: nodes, Stratum, kHeavyHash hardware, and where to read authoritative specs—without profit promises."
---

# From a hobby node to solo mining — a gentle map

> [!NOTE]
> Mining economics change with difficulty, electricity cost, hardware price, and market conditions. This article is a **conceptual map** and points to technical documentation; it is **not** investment or tax advice, and it does **not** guarantee that solo mining is profitable or appropriate in every case. Always verify **official node, bridge, and miner software** against current project releases before deploying real hashrate or funds.

Teams and individuals running—or considering—a **full node** often ask what it takes to **mine Zorkcoin (ZORK)** directly instead of only validating others’ blocks. On **Zork Network**, proof-of-work uses **kHeavyHash**, and mining hardware typically speaks to infrastructure through **Stratum**-style protocols. The jump from “node online” to “block found” is not a single button; it is a **chain of components** that must agree on templates, difficulty, and block headers. This page sketches that chain in plain terms so planning and verification can happen before capital is committed.

---

## Terms in one place

- **Zork Network** — the system.
- **Zorkcoin / ZORK** — the currency and ticker.
- **Full node** — software that fully validates blocks and transactions and follows the network’s consensus rules (including kHeavyHash and the Zorkcoin header rules described in [Running a Full Node on Zork Network](Running_A_Full_Node.md)).
- **Pool mining** — your miner submits **shares** to an operator who combines work from many machines and pays out participants by a published scheme.
- **Solo mining** — you (or your own stack) attempt to produce blocks that meet **network difficulty** without routing rewards through a pool’s accounting. Variance is high: rewards can be infrequent or clustered compared with steady small pool payouts.
- **Stratum** — a common JSON-RPC-based protocol for delivering work to miners and receiving shares or block candidates. This site’s [kHeavyHash Stratum Protocols](../technical-details/kHeavyHash_Stratum_Protocols.md) document describes the variant used with kHeavyHash ASICs and compatible software, including **Zorkcoin**.

---

## Why a node sits at the center

Whether mining is done through pools or solo infrastructure, **someone** must build **valid block templates**: choose transactions, respect consensus rules, and serialize headers so the proof-of-work puzzle is well-defined. Public pools run that logic at scale. A solo setup usually means the operator, through a local node and trusted bridge software, takes responsibility for that template path.

Running a local node is therefore not just ideological—it is often **structural** for solo operation: the template source should match the same rules enforced by the mining stack, with a clear path from “template” to “header bytes” to “kHeavyHash input.” The exact RPC or API names depend on the **Zork Network** full-node implementation in use; this documentation site does not duplicate those manuals. The node’s **official documentation** remains authoritative for commands and configuration.

Mining only through a remote pool and never running a local node means relying on the pool’s view of chain state and work construction. Many miners accept that trade-off for simplicity. Solo setups trade convenience for **direct alignment** between local validation rules and block production, at the cost of operational complexity.

---

## The usual shape of a solo stack

Think in layers:

1. **Consensus and templates** — A synced full node (or equivalent) exposes block templates consistent with Zorkcoin rules: **kHeavyHash** proof-of-work, **64-bit** nonce and timestamp fields in the header, and **millisecond**-resolution timestamp semantics, as summarized in the full-node article linked above.
2. **Stratum endpoint** — Consumer mining hardware and most mining software expect a **Stratum server**. In a solo arrangement, a **bridge**, **proxy**, or **pool software** in solo mode often sits between the node and the miner: it requests templates, converts them into Stratum `mining.notify` work, and accepts submissions.
3. **Miner** — ASIC (for example kHeavyHash units discussed in [kHeavyHash Miners](../information/kHeavyHash_Miners.md)) or GPU software that implements the client side of Stratum and performs the hash search.

The [kHeavyHash Stratum Protocols](../technical-details/kHeavyHash_Stratum_Protocols.md) specification explains, from both server and client perspectives, how work is encoded—**Standard Format** versus **BigJob Format**, extranonce behavior, difficulty checks, and error codes. The companion [KHeavyHash Stratum Protocol Specification](../technical-details/kheavyhash_stratum_prot_part2.md) document states that it describes the protocol **as implemented by the Zorkcoin Stratum Bridge**—useful context when you evaluate which software stack actually supports **Zorkcoin** headers end-to-end.

**Critical accuracy point:** public examples in generic kHeavyHash documentation sometimes reference **third-party bridges** written for other blockchains (for illustration only). Operational deployments need software that speaks the selected node’s template API **and** the **Zorkcoin** Stratum rules. A bridge built for another coin should not be treated as plug-and-play without explicit Zork Network support and release notes.

---

## Difficulty, time, and expectations

Network difficulty adjusts so blocks arrive near a target rate. Zorkcoin’s adjustment algorithm is documented in [Zorkcoin™ Difficulty Adjustment Algorithm](../technical-details/Zorkcoin_Difficulty_Adjustment_Algorithm.md). As a miner, you care because **network difficulty** sets the bar for a **valid block**, while **Stratum difficulty** often sets a lower bar for **shares** (useful for pools and for monitoring hashrate).

Solo miners still care about share difficulty when using Stratum infrastructure locally: the bridge or pool software uses shares to estimate progress and to validate that miners are hashing the intended work, even when only **block solutions** pay rewards.

**Expectation management:** higher variance is normal for solo mining at fixed hashrate. Pool mining smooths payouts by statistics; solo mining does not. Whether that trade-off makes sense depends on electricity pricing, hardware cost, noise constraints, and tolerance for long dry spells. Verify assumptions with current data and official release documentation rather than relying on any single article.

---

## Hardware and software: where this site helps

This documentation site will not recommend a specific commercial pool or hosted service. It **will** help you understand the landscape:

- **[kHeavyHash Miners](../information/kHeavyHash_Miners.md)** surveys manufacturers and models believed accurate when written, with reminders that pricing and specs drift—verify before purchasing.
- **[kHeavyHash Stratum Protocols](../technical-details/kHeavyHash_Stratum_Protocols.md)** is the deep reference for message formats and mining flow.
- **[kHeavyHash Technical Overview](../technical-details/kHeavyHash_Technical_Overview.md)** (linked from the Stratum doc) covers the algorithm itself.

GPU miners listed in the Stratum overview (for example tools with Kaspa or kHeavyHash compatibility) illustrate how **software** fits into the ecosystem; **compatibility with Zorkcoin’s node and bridge stack** must be confirmed for your exact versions. Prefer **official** or **maintainer-documented** combinations over forum hearsay.

---

## Security and operational habits

Solo setups touch **funds** and **network exposure**:

- **Run verified binaries** from known releases, and verify hashes or signatures when the project provides them.
- **Limit RPC exposure** on your node: bridges often need local access; avoid exposing privileged RPC to the public Internet without hardening you fully understand.
- **Wallet and payout paths** — Understand where block rewards land (native wallet, configured address, cold storage). Misconfiguration here is a common source of “I found a block but where is the coin?” confusion.
- **Monitoring** — Disk space, peer count, clock skew, and bridge logs matter when templates stall or submissions fail.

These habits align with treating Zorkcoin as a **financial tool** first: predictable, understandable behavior beats rushing online with default passwords and open ports.

---

## A learning path you can actually follow

If you want to **explore** before committing capital:

1. Read [Running a Full Node on Zork Network](Running_A_Full_Node.md) and bring a node to a healthy synced state on testnet or mainnet, per official instructions.
2. Skim [kHeavyHash Stratum Protocols](../technical-details/kHeavyHash_Stratum_Protocols.md) sections on connection establishment, `mining.notify`, and difficulty checking—enough to recognize what your miner and bridge are doing.
3. Read [Zorkcoin™ Difficulty Adjustment Algorithm](../technical-details/Zorkcoin_Difficulty_Adjustment_Algorithm.md) once, so difficulty changes feel less mysterious when you watch the chain.
4. When you are ready for hardware decisions, use [kHeavyHash Miners](../information/kHeavyHash_Miners.md) as a starting list, then confirm everything with vendors and your power budget.

If you discover that documentation here conflicts with the latest release, **open an issue** on [GitHub](https://github.com/ZorkNetwork/docs.zork.network/issues). Accurate maps only stay accurate when readers report drift.

---

## Ways to contribute if you stop short of solo mining

Not everyone will solo-mine—and that is fine. You can still advance the ecosystem by improving documentation, translations, or node operation. See [Ways to Support Zork Network Without Mining](Ways_To_Support_Zork_Network_Without_Mining.md) for issue filing, pull requests, and locale workflows.

---

## Summary

- Solo mining on Zork Network is a **system**: **node → template/bridge → Stratum → miner**, all speaking the same consensus and protocol dialect.
- **kHeavyHash** and **Zorkcoin-specific header rules** mean you must pair **Zorkcoin-aware** software, not guess from other coins’ tutorials.
- **Variance and economics** dominate the “should I?” question; this site maps **how**, not whether, solo mining deserves your capital.
- **Official project documentation** for the node and bridge remains the final word on RPC names, ports, and supported miner versions.

When you treat mining as engineering rather than speculation, you are already closer to **understanding your money**—and to helping others do the same with patience and precision.
