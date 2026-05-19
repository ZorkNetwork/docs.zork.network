---
layout: default
title: Legacy Soft Forks Inherited at Genesis
nav_order: 7
lang: en
---

# Legacy Soft Forks Inherited at Genesis

This page explains the names you often see on block explorers—BIP numbers, CSV, SegWit, and similar lines—and what each group of rules actually does. On older Bitcoin-style chains those lines usually record soft forks: changes that tightened validation over time without breaking older software completely, announced with activation heights and miner signaling until the rule went live everywhere. Understanding the labels removes a lot of the mystery next to ticker symbols or wallet dialogs.

From the first block, Zork Network already enforces the same legacy soft-fork bundle. There is nothing to “switch on” later for these items: wallets, nodes, and explorers still say “BIP34” or “SegWit” because the codebase and tooling share wording with upstream Bitcoin and Litecoin, not because activation is pending.

The headings follow the usual explorer buckets for those historical forks. Taproot and MimbleWimble Extension Blocks (optional private transfers, often abbreviated MWEB) landed later on Litecoin’s timeline; see [Legacy Soft Forks Enabled Post-Genesis](Legacy_Soft_Forks_Enabled_Post_Genesis.md) for the outline and todo list for documenting them.

## BIP34 - Block Height in Coinbase (Safer Block Versioning)

BIP34 requires miners to include the block height in the coinbase transaction input script. In simple terms, every block carries an explicit height marker that full nodes can check while validating. This improves block uniqueness and reduces ambiguity that could appear in older edge cases where coinbase data was less structured.

BIP34 also helped formalize block version transitions in a way nodes could enforce at consensus level. That sounds like an internal maintenance detail, but it has practical value: upgrades are easier to reason about when node software can validate clear structural expectations in block data. Users usually see this indirectly as more predictable network behavior over long periods.

On Zork Network, treating BIP34 behavior as inherited baseline consensus means tooling does not need to split logic between a pre-activation era and a post-activation era for this rule. Explorers, parsers, and analytics code can assume height-in-coinbase semantics consistently across chain history, which simplifies education and troubleshooting.

## BIP65 - CHECKLOCKTIMEVERIFY (Time Locks at Script Level)

BIP65 introduced `CHECKLOCKTIMEVERIFY` (CLTV), a script opcode for absolute locktimes. It allows script paths that are valid only after a specific block height or timestamp threshold is reached. This makes it possible to express time-based spending policies directly in consensus script, not just in wallet-side conventions.

The practical benefit is safer contract design. Without CLTV, many time-dependent constructions require fragile workarounds or looser trust assumptions. With CLTV, developers can build clear "not spendable until X" conditions for use cases such as delayed recovery, staged payouts, escrow variants, and other policies where early spending must fail at consensus.

For Zork Network users, inherited CLTV support means modern timelock scripting is a baseline feature rather than a late-era add-on. For newcomers, the key takeaway is that the chain supports mature, enforceable time-based script conditions from the start, which broadens what secure transaction policies can be built.

## BIP66 - Strict DER Signatures (Canonical Signature Encoding)

BIP66 enforces strict DER encoding for ECDSA signatures during script validation. In plain language, signatures must follow one canonical byte-level structure instead of permitting loose or non-standard encodings. This removes parser ambiguity and helps prevent cases where one implementation accepts a signature that another rejects.

Consensus systems depend on many independent nodes reaching the same validation outcome for the same transaction bytes. If signature encoding rules are permissive, implementation differences can become consensus risk. BIP66 narrows that risk by requiring strict, predictable encoding and reducing accidental divergence between clients.

On Zork Network, this inherited rule means strict signature-format discipline is part of normal validation expectations. Developers integrating signing logic for legacy ECDSA paths should assume canonical DER requirements, and users benefit from fewer cross-implementation surprises in transaction acceptance behavior.

## CSV Activation Soft Fork (BIP68, BIP112, BIP113)

CSV is shorthand for a coordinated soft-fork package that historically grouped related transaction-finality improvements: sequence-based relative locktime semantics, script-level enforcement for those sequence locks, and a safer reference clock for time-based lock decisions. Although these proposals solve different technical problems, they reinforce each other operationally.

Historically, it made sense to activate these changes together. BIP68 defines how `nSequence` can encode relative constraints, BIP112 exposes that capability in script via `CHECKSEQUENCEVERIFY`, and BIP113 makes locktime checks rely on median-time-past rather than a miner-chosen block timestamp. Bundling them reduced partial-state complexity and aligned locktime behavior across mempool and block validation contexts.

The key model is that CSV-era behavior is inherited as part of the chain's consensus foundation. Even when tools mention historical activation language from Bitcoin-family ancestry, these can be interpreted as baseline semantics on Zork when evaluating wallet behavior, script conditions, and transaction eligibility.

### BIP68 - Relative Lock-Time Using `nSequence`

BIP68 gives consensus meaning to parts of each input's `nSequence` field (for transaction version 2+). Instead of only serving as a non-finality marker, sequence values can express relative age constraints tied to the spent output. That means a transaction input can be invalid until enough blocks have passed, or enough relative time has elapsed, after the UTXO being spent was confirmed.

The design includes two major modes: block-based relative locks and time-based relative locks. Time-based sequence locks use coarse 512-second units, while block-based locks count block intervals. The BIP also defines disable/type bits and masking rules so implementations interpret the field consistently and preserve room for future extension.

For newcomers, the practical effect is that some transactions are intentionally "not yet valid" based on how old their inputs are, not just based on an absolute date in `nLockTime`. This is useful for channels and other staged protocols, and is easier to reason about when relative locks are understood as input-age constraints rather than wall-clock time alone.

### BIP112 - CHECKSEQUENCEVERIFY (Script Access to Relative Locks)

BIP112 redefines `OP_NOP3` as `CHECKSEQUENCEVERIFY` (CSV), giving scripts a way to enforce relative lock conditions against transaction sequence values. In plain terms, BIP68 provides the consensus lock semantics, and BIP112 lets script authors actually use those semantics in spending paths.

This matters for contract construction. A script can require that a spend branch only becomes valid after a relative delay, while another branch stays immediately spendable under different conditions. That pattern is central to many advanced constructions, including channel safety mechanisms and timeout-based recovery branches that need enforceable response windows.

Operationally, BIP112 reduces reliance on policy assumptions by moving relative-delay guarantees into script validation itself. On Zork Network, inherited CSV script behavior means developers can treat this opcode as part of the normal toolkit for relative timelock logic, rather than a compatibility edge case that only applies after a historical fork height.

### BIP113 - Median Time Past (Safer Time Endpoint for Locktime Checks)

BIP113 changes how time-based lock conditions are evaluated: instead of using the candidate block's header timestamp directly, validation uses median time past (MTP) from recent blocks. This provides a more monotonic and manipulation-resistant time reference for consensus lock decisions.

The motivation is straightforward: block timestamps have slack, and miners could otherwise gain by timestamping blocks in ways that make time-locked transactions appear valid earlier. MTP reduces that incentive by grounding locktime checks in a rolling median over prior blocks, which is harder to skew in one block and behaves more predictably.

For users and integrators, this means time-based locks may confirm later than naive wall-clock expectations, but in exchange the network gets more consistent locktime semantics. In combination with BIP68 and BIP112, BIP113 helped make transaction finality and timelock behavior more coherent, which is why it was historically activated in the same soft-fork bundle.

## Segregated Witness (SegWit) Activation and Supporting BIPs

A complete SegWit-era reference set includes BIP141, BIP143, BIP144, BIP147, BIP91, BIP148, and BIP173. BIP147 is part of that set because it closes a related malleability vector and was deployed with the SegWit consensus package in Bitcoin-family history.

At a high level, SegWit separates signature and witness data from the pre-SegWit transaction serialization used for `txid`, introduces new witness program rules, and improves transaction malleability behavior for supported spend types. That change also enabled cleaner script evolution and improved weight accounting compared with the previous single-size model. The result is a package that touches consensus rules, signature hashing rules, relay serialization, and address standards.

Historically, these BIPs were not all the same type of change. Some define consensus behavior (such as BIP141 and BIP143), some define peer-to-peer transport details (BIP144), some describe activation strategies used during Bitcoin's SegWit deployment period (BIP91 and BIP148), and one standardizes user-facing address encoding (BIP173). On Zork Network, inherited behavior means these semantics are treated as baseline assumptions by tools, while activation history remains useful technical context.

### BIP141 - Segregated Witness (Consensus Layer)

BIP141 is the core SegWit consensus specification. It defines witness programs, moves witness data outside the legacy transaction serialization used for `txid`, and introduces witness commitments in blocks. In practical terms, this is the rule set that makes SegWit "real" at consensus level rather than just a wallet convention.

It also introduces the concept of block weight, which blends base data and witness data into a single limit model. This changed how capacity is measured and helped avoid hard-fork-style block-size jumps while still allowing more efficient use of block space. For users, the visible effects are usually better fee efficiency for SegWit spends and reduced third-party malleability in supported transaction flows.

For Zork Network readers, BIP141 is best understood as the anchor specification in this family: other SegWit-era BIPs either support it directly, make it safer to use, or improve how it is relayed and represented. If a tool says "SegWit active," BIP141 semantics are the core behavior it is referring to.

### BIP143 - New Signature Hash for Version 0 Witness Programs

BIP143 defines a new digest algorithm for signature verification of version 0 witness programs (for example, P2WPKH and P2WSH paths). It changes what data is committed during signing so validation is both safer and more efficient than legacy sighash handling in several important cases.

One practical improvement is clearer commitment to input amounts in the signed message, which helps hardware signing flows and reduces ambiguity in offline signing contexts. Another is performance: by restructuring hashing for SegWit v0, implementations can avoid some expensive recomputation patterns that existed in older signing logic.

For new users, the shortest explanation is: BIP141 creates the SegWit transaction model, and BIP143 defines how signatures are computed inside that model for v0 witness programs. Wallets and signing libraries that support SegWit v0 depend on these rules even if users never see "BIP143" in the interface.

### BIP144 - Segregated Witness Peer Services (P2P Relay)

BIP144 specifies how SegWit-aware nodes announce support and exchange witness-bearing transactions and blocks over peer-to-peer networking. It defines new serialization behavior and inventory/getdata conventions so upgraded nodes can transfer witness data while keeping backward compatibility with older nodes.

This is a transport-layer companion to consensus-layer SegWit. Without BIP144-style relay behavior, nodes might validate SegWit rules in theory but fail to propagate witness-rich data efficiently in practice. The BIP introduces `NODE_WITNESS` signaling and witness-aware message handling that let peers coordinate capabilities cleanly.

For newcomers, BIP144 is easiest to remember as "the networking glue" for SegWit. BIP141 says what is valid, while BIP144 helps nodes actually move that data around the network in a compatible way.

### BIP91 - Reduced-Threshold Miner-Activated Signaling for SegWit

BIP91 was a coordination soft fork used during Bitcoin's SegWit activation period. It let a miner majority enforce SegWit signaling rules on blocks before the original higher-threshold deployment had naturally locked in. It was designed as a compatibility bridge, not as a replacement for the SegWit consensus rules themselves.

Technically, BIP91 required blocks to signal for the existing SegWit deployment while BIP91 conditions were active, helping converge miner behavior and avoid indefinite stalemate. This was part of a very specific historical activation window and reflected governance pressure as much as protocol engineering.

For Zork Network documentation readers, BIP91 is primarily historical context: it explains why older discussions, charts, and tools reference multiple activation paths for the same SegWit rule set. It is less about day-to-day transaction mechanics and more about deployment history.

### BIP148 - User-Activated Mandatory SegWit Signaling (UASF Path)

BIP148 proposed a user-activated soft-fork path that would reject non-signaling blocks during a defined time window if SegWit had not already locked in. Like BIP91, this was an activation strategy around the existing SegWit deployment, not a new set of witness consensus rules.

Its importance is social as well as technical. BIP148 demonstrated how economically relevant full nodes can coordinate enforcement preferences even when miner signaling is delayed. That made it a landmark in soft-fork governance discussions across Bitcoin-family communities.

For practical understanding, BIP148 helps explain references to "UASF" in historical SegWit material. On a chain where inherited SegWit behavior is already baseline, BIP148 is mainly valuable as background on how activation disputes were resolved, not as a rule users need to configure today.

### BIP173 - Bech32 Address Format for Native Witness Outputs

BIP173 defines Bech32, a checksummed base32 address format for native witness outputs. Compared with older Base58 address formats, it is designed for better error detection, improved readability, and better QR efficiency. This is why many SegWit addresses are human-readable prefixes plus `1...` data payloads in lowercase.

Importantly, BIP173 is an application-layer/address standard, not the consensus rule that created SegWit itself. It standardizes how wallets encode witness destinations so users can send funds to native witness outputs more safely and interoperably. In historical deployment terms, this made SegWit easier to use correctly in real wallets.

For readers of Zork docs, BIP173 is the user-facing part most people actually touch: address strings, scanning QR codes, and validating destination formats. Even when people never read BIP141, they often interact with BIP173-derived behavior every time they copy a SegWit-style address.

### BIP147 - NULLDUMMY for `CHECKMULTISIG` Malleability Fix

BIP147 addresses a malleability vector involving the extra dummy stack element consumed by `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY`. It enforces the `NULLDUMMY` rule so that this element must be empty, eliminating a mutation path that could otherwise alter transaction identifiers in relevant contexts.

This BIP was deployed with the SegWit package in Bitcoin history and is part of the broader malleability-hardening story. While SegWit substantially improves malleability behavior, BIP147 closes a specific remaining script-level edge case tied to multisig opcode behavior.

For Zork Network readers, the takeaway is that BIP147 belongs in the complete SegWit-era map even though it is less famous than BIP141 or BIP173. Including it helps explain why mature transaction validation stacks reference several companion fixes, not just one headline feature.
