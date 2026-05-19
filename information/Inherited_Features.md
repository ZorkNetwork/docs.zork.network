---
layout: default
title: Inherited Features
nav_order: 9
lang: en
published: false
nav_exclude: true
search_exclude: true
hide: true
---

# Inherited Features

Zork Network inherits behavior from Bitcoin-family software beyond **consensus** rules (what makes a block or transaction valid chain-wide). Peer-to-peer conventions, address formats, historical activation coordination, and Bloom-filter advertisement also show up in logs, wallets, and node configuration. This page collects those **non-consensus** inherited items that are commonly labeled with Bitcoin Improvement Proposal numbers.

**Version-bit soft-fork deployment ([BIP9](https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki))** is documented under [Legacy Soft Forks Inherited at Genesis](Legacy_Soft_Forks_Inherited_at_Genesis.md): full nodes enforce deployment states (STARTED, LOCKED_IN, ACTIVE, FAILED) in consensus code, which determines **when** new validity rules apply chain-wide.

Consensus-only validation rules—including SegWit witness commitments, CSV semantics, P2SH evaluation, duplicate-transaction prevention, and related script soft forks—are summarized in [Legacy Soft Forks Inherited at Genesis](Legacy_Soft_Forks_Inherited_at_Genesis.md).

## Inherited peer relay, addresses, and activation history

The subsections below do **not** replace SegWit consensus rules ([BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki)) by themselves. They standardize how witness-related data moves on the wire (BIP144), how SegWit activation was coordinated on Bitcoin (BIP91, BIP148), how native witness destinations are encoded for users (BIP173), and how light-client bloom service is advertised (BIP111, with BIP37 referenced).

### BIP111 - NODE_BLOOM Service Bit (Explicit Bloom-Filter Support)

[BIP111](https://github.com/bitcoin/bips/blob/master/bip-0111.mediawiki) extends [BIP37](https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki) connection bloom filtering by defining the **`NODE_BLOOM`** service bit (`1 << 2`) so peers can advertise support for bloom-filter-based relay explicitly. The reference implementation also bumped protocol version so newer peers can distinguish behavior from older nodes that accepted filter messages without advertising the bit.

The motivation is operational and privacy-related: unfettered bloom-filter service can be a **resource burden** on full nodes and offers weak privacy for light clients; operators benefit from being able to disable offering that service while remaining otherwise helpful peers. Nodes that do not support bloom filtering should disconnect peers that send `filterload`, `filteradd`, or `filterclear` after the compatibility rules described in the BIP.

For Zork Network documentation readers, BIP111 is peer-layer vocabulary, not a consensus opcode change. Light-client authors care whether peers advertise `NODE_BLOOM`; full-node operators care whether they wish to serve bloom filters at all. Together with BIP37 semantics, it completes the picture of how SPV-style filtering was standardized and later constrained on Bitcoin-family networks.

### BIP144 - Segregated Witness Peer Services (P2P Relay)

[BIP144](https://github.com/bitcoin/bips/blob/master/bip-0144.mediawiki) specifies how SegWit-aware nodes announce support and exchange witness-bearing transactions and blocks over peer-to-peer networking. It defines new serialization behavior and inventory/getdata conventions so upgraded nodes can transfer witness data while keeping backward compatibility with older nodes.

This is a transport-layer companion to consensus-layer SegWit ([BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki), documented under consensus features). Without BIP144-style relay behavior, nodes might validate SegWit rules in theory but fail to propagate witness-rich data efficiently in practice. The BIP introduces `NODE_WITNESS` signaling and witness-aware message handling that let peers coordinate capabilities cleanly.

For newcomers, BIP144 is easiest to remember as the networking complement to SegWit consensus rules: BIP141 defines validity; BIP144 defines how peers move witness-bearing messages on the wire.

### BIP91 - Reduced-Threshold Miner-Activated Signaling for SegWit

[BIP91](https://github.com/bitcoin/bips/blob/master/bip-0091.mediawiki) was a coordination soft fork used during Bitcoin's SegWit activation period. It let a miner majority enforce SegWit signaling rules on blocks before the original higher-threshold deployment had naturally locked in. It was designed as a compatibility bridge, not as a replacement for the SegWit consensus rules themselves.

Technically, BIP91 required blocks to signal for the existing SegWit deployment while BIP91 conditions were active, helping converge miner behavior and avoid indefinite stalemate. This was part of a very specific historical activation window and reflected governance pressure as much as protocol engineering.

For Zork Network documentation readers, BIP91 is primarily historical context: it explains why older discussions, charts, and tools reference multiple activation paths for the same SegWit rule set. It is less about day-to-day transaction mechanics and more about deployment history.

### BIP148 - User-Activated Mandatory SegWit Signaling (UASF Path)

[BIP148](https://github.com/bitcoin/bips/blob/master/bip-0148.mediawiki) proposed a user-activated soft-fork path that would reject non-signaling blocks during a defined time window if SegWit had not already locked in. Like BIP91, this was an activation strategy around the existing SegWit deployment, not a new set of witness consensus rules.

Its importance is social as well as technical. BIP148 demonstrated how economically relevant full nodes can coordinate enforcement preferences even when miner signaling is delayed. That made it a landmark in soft-fork governance discussions across Bitcoin-family communities.

For practical understanding, BIP148 helps explain references to "UASF" in historical SegWit material. On a chain where inherited SegWit behavior is already baseline, BIP148 is mainly valuable as background on how activation disputes were resolved, not as a rule users need to configure today.

### BIP173 - Bech32 Address Format for Native Witness Outputs

[BIP173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki) defines Bech32, a checksummed base32 address format for native witness outputs. Compared with older Base58 address formats, it is designed for better error detection, improved readability, and better QR efficiency. This is why many SegWit addresses are human-readable prefixes plus `1...` data payloads in lowercase.

Importantly, BIP173 is an application-layer address standard, not the consensus rule that created SegWit itself. It standardizes how wallets encode witness destinations so users can send funds to native witness outputs more safely and interoperably. In historical deployment terms, this made SegWit easier to use correctly in real wallets.

For readers of Zork docs, BIP173 is the user-facing part most people actually touch: address strings, scanning QR codes, and validating destination formats. Even when people never read BIP141, they often interact with BIP173-derived behavior every time they copy a SegWit-style address.

## Candidate topics for future expansion

Community summaries of Bitcoin Core history—such as answers under [What BIPs are supported by the standard client Bitcoin Core?](https://bitcoin.stackexchange.com/questions/18851/what-bips-are-supported-by-the-standard-client-bitcoin-core)—often mix consensus rules with wallet standards, RPC APIs, and peer-layer messages. This page holds **non-consensus** candidates for later write-ups. Consensus-only candidates remain on [Legacy Soft Forks Inherited at Genesis](Legacy_Soft_Forks_Inherited_at_Genesis.md).

- **BIP11** — Standard `M-of-N` multisignature output patterns (frequently used with P2SH).
- **BIP13** — Address format for P2SH (pairs directly with [BIP16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki); consensus evaluation is documented under consensus features).
- **BIP14** — Peer **subversion / user-agent** conventions for capability signaling.
- **BIP21** — **URI scheme** for payment requests (`bitcoin:`-style links; chain-specific variants apply).
- **BIP22 / BIP23** — **`getblocktemplate`** RPC mining protocol and extensions (solo/pool integration context).
- **BIP31** — **`pong`** message and related protocol version behavior for connection health.
- **BIP32** — **Hierarchical deterministic (HD) wallets** and key-derivation conventions.
- **BIP35** — **`mempool`** P2P message for mempool inspection/query patterns.
- **BIP37** — **Bloom-filter-based SPV relay** (complements BIP111; privacy and DoS considerations).
- **BIP61** — **`reject`** P2P message for relay diagnostics between peers.
- **BIP70 / BIP71 / BIP72** — Payment Protocol stack for invoice-style payments (wallet and merchant integration; deployment varies by ecosystem).
- **BIP125** — **Opt-in replace-by-fee** mempool signaling semantics.
- **BIP130** — **`sendheaders`** for more efficient header announcements between peers.
- **BIP133** — **`feefilter`** messages for fee-based relay coordination.
- **BIP152** — **Compact blocks** and related block relay optimizations.

For an implementation-maintained inventory rather than a static Q&A thread, see Bitcoin Core’s [`doc/bips.md`](https://github.com/bitcoin/bitcoin/blob/master/doc/bips.md) (upstream reference only; Zorkcoin inherits selectively from Litecoin/Bitcoin-family lineage).




## Other Early Features

Deployment Signaling, Script Packaging, Chain Integrity, and Peer Advertising

The following items often appear alongside BIP34-style consensus rules in Bitcoin-family documentation: how soft forks are scheduled and signaled (BIP9), how complex scripts are funded via short hashes (BIP16), how duplicate transaction identifiers are forbidden while unspent outputs remain (BIP30), and how nodes advertise support for bloom-filter-based SPV queries (BIP111). Together they illustrate how consensus fixes, script ergonomics, and peer-layer behavior evolved as a package over years on Bitcoin and Litecoin, then became baseline assumptions for descendant chains such as Zork Network.

Full nodes enforce **when** a consensus soft fork becomes active through deployment state machines—not only the script or block rules that apply afterward. [BIP9](https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki) specifies that scheduling machinery for Bitcoin-class clients: miners signal with bits in `nVersion`; after lock-in and delay, new validity rules apply chain-wide. That coupling of signaling and activation is consensus-critical infrastructure.

### BIP9 - Version Bits with Timeout and Delay (Parallel Soft-Fork Signaling)

[BIP9](https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki) defines a deployment framework for backward-compatible (soft-fork) changes using bits in the block `nVersion` field, measured over difficulty retarget periods, with defined states from deployment start through lock-in, activation, or failure after a timeout. Multiple deployments can use distinct bits within the allowed range, subject to scheduling rules, which replaced the older pattern of tying each upgrade to monotonically increasing version integers alone.

Implementations track those states in consensus code: once a deployment reaches **ACTIVE**, the associated soft-fork rules are enforced for subsequent blocks. Many upgrades on Bitcoin—including CSV (BIP68/112/113) and SegWit—were rolled out with version bits or closely related signaling baked into the same consensus layer. The exact parameters (bit index, start time, timeout, threshold) are chain-specific and live in each network’s consensus parameters rather than in the BIP text alone.

On Zork Network, inherited full-node behavior may still interpret or document version bits when discussing upgrades or miner signaling, even when individual deployments from Bitcoin’s timeline are only historical reference. Treat BIP9 as the **pattern** for staged soft-fork rollout: predictable states, miner-visible signaling, and eventual activation or clean failure—not as a single fixed schedule across all chains.

### BIP16 - Pay to Script Hash (P2SH)

[BIP16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki) introduces **pay-to-script-hash**: outputs that commit to `OP_HASH160 <20-byte-script-hash> OP_EQUAL`, while the full redeem script is supplied only when the output is spent. Senders present a fixed-size hash to the chain; redeemers supply the script and satisfaction data. That division made multisignature and other non-trivial scripts practical for everyday wallets and exchanges without forcing senders to handle long or fragile script payloads.

Validation performs an initial script execution, checks that the pushed script hashes to the committed value, then runs the deserialized script as the effective `scriptPubKey` with the remaining stack. Signature-operation counting rules apply to the redeem script so block-wide DoS limits remain enforceable. [BIP13](https://github.com/bitcoin/bips/blob/master/bip-0013.mediawiki) defines the Base58 address encoding for P2SH; operationally, “P2SH address” and “BIP16 output type” refer to the same consensus pattern.

For Zork Network participants, P2SH remains part of the inherited script vocabulary alongside native SegWit and other paths documented elsewhere. Wallets and integrators should treat P2SH as a standard way to lock funds to **policy encoded by hash**, not merely as a legacy curiosity—while still preferring modern native Segwit/Taproot-style receives where the project recommends them for efficiency and features.

### BIP30 - Duplicate Transactions (No Reusing an Active txid)

[BIP30](https://github.com/bitcoin/bips/blob/master/bip-0030.mediawiki) adds a consensus rule: a block must not include a transaction whose identifier (`txid`) matches an **earlier transaction in the same chain that still has unspent outputs**. Duplicates were once theoretically exploitable in ways that broke assumptions about confirmation safety and reorg handling; the rule closes that class of problems without requiring globally unique txids for fully spent historical transactions (allowing duplication after full spend supports future pruning models).

Bitcoin grandfathered two historical violating blocks; other chains following the same rule set may have analogous carve-outs or none, depending on their genesis and history. The underlying principle is stable: **the spendable UTXO set must not change in surprising ways** when blocks are connected and disconnected during reorganizations.

For Zork Network validation stacks inherited from Litecoin-family code, BIP30-style semantics are part of why transaction identities and coinbase uniqueness assumptions behave like mature Bitcoin-class networks. Application developers rarely toggle this behavior; it is foundational integrity plumbing that keeps mempool and chain processing predictable.

### BIP111 - NODE_BLOOM Service Bit (Explicit Bloom-Filter Support)

[BIP111](https://github.com/bitcoin/bips/blob/master/bip-0111.mediawiki) extends [BIP37](https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki) connection bloom filtering by defining the **`NODE_BLOOM`** service bit (`1 << 2`) so peers can advertise support for bloom-filter-based relay explicitly. The reference implementation also bumped protocol version so newer peers can distinguish behavior from older nodes that accepted filter messages without advertising the bit.

The motivation is operational and privacy-related: unfettered bloom-filter service can be a **resource burden** on full nodes and offers weak privacy for light clients; operators benefit from being able to disable offering that service while remaining otherwise helpful peers. Nodes that do not support bloom filtering should disconnect peers that send `filterload`, `filteradd`, or `filterclear` after the compatibility rules described in the BIP.

For Zork Network documentation readers, BIP111 is peer-layer vocabulary, not a consensus opcode change. Light-client authors care whether peers advertise `NODE_BLOOM`; full-node operators care whether they wish to serve bloom filters at all. Together with BIP37 semantics, it completes the picture of how SPV-style filtering was standardized and later constrained on Bitcoin-family networks.

## Candidate topics for future expansion

Community summaries of Bitcoin Core history—such as answers under [What BIPs are supported by the standard client Bitcoin Core?](https://bitcoin.stackexchange.com/questions/18851/what-bips-are-supported-by-the-standard-client-bitcoin-core)—often inventory wallet, RPC, and peer-layer BIPs alongside consensus rules. This page focuses on inherited consensus and closely related deployment or peer semantics already discussed above. The items below are **not yet summarized here** but are strong candidates for future sections or linked deep-dives, especially for wallet, mining, and node-operation audiences:

- **BIP11** — Standard `M-of-N` multisignature output patterns (frequently used with P2SH).
- **BIP13** — Address format for P2SH (pairs directly with BIP16).
- **BIP14** — Peer **subversion / user-agent** conventions for capability signaling.
- **BIP21** — **URI scheme** for payment requests (`bitcoin:`-style links; chain-specific variants apply).
- **BIP22 / BIP23** — **`getblocktemplate`** RPC mining protocol and extensions (solo/pool integration context).
- **BIP31** — **`pong`** message and related protocol version behavior for connection health.
- **BIP32** — **Hierarchical deterministic (HD) wallets** and key-derivation conventions.
- **BIP35** — **`mempool`** P2P message for mempool inspection/query patterns.
- **BIP37** — **Bloom-filter-based SPV relay** (complements BIP111; privacy and DoS considerations).
- **BIP42** — Consensus fix ensuring the **subsidy schedule terminates** as intended (supply-schedule correctness).
- **BIP61** — **`reject`** P2P message for relay diagnostics between peers.
- **BIP70 / BIP71 / BIP72** — Payment Protocol stack for invoice-style payments (wallet and merchant integration; deployment varies by ecosystem).
- **BIP125** — **Opt-in replace-by-fee** mempool signaling semantics.
- **BIP130** — **`sendheaders`** for more efficient header announcements between peers.
- **BIP133** — **`feefilter`** messages for fee-based relay coordination.
- **BIP152** — **Compact blocks** and related block relay optimizations.

For an implementation-maintained inventory rather than a static Q&A thread, see Bitcoin Core’s [`doc/bips.md`](https://github.com/bitcoin/bitcoin/blob/master/doc/bips.md) (upstream reference only; Zorkcoin inherits selectively from Litecoin/Bitcoin-family lineage).
