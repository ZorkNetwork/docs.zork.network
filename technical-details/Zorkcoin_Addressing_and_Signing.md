---
layout: default
title: Zorkcoin™ Addressing & Signing
nav_order: 100
---

# Zorkcoin™ Addressing & Signing

```{note}
This is a work in progress.  This note will be removed after a proofreading.  Feel free to contribute by asking questions where further clarification is required, or by providing constructive feedback.
```

## Overview

Addressing is basically the same as Bitcoin and Litecoin other than the
identifiers being changed to indicate Zorkcoin™ to avoid 'reuse'.

## Legacy

## Segregated Witness

## Taproot

## R&D In Progress

### Post Quantum Cryptography (PQC) / Falcon



---

**Quick summary:** **Zorkcoin uses Litecoin-style address schemes but with distinct network prefixes; the network expects modern Taproot (P2TR) and MWEB addresses as the primary receive formats, with legacy P2WPKH/P2SH/P2PKH supported for compatibility**. @todo: confirm exact Zorkcoin prefix bytes in the repository.

### Overview
Zorkcoin is a code fork of Litecoin and inherits the same address mechanisms (Base58Check and Bech32-style encodings) and the same address semantics (pubkey-hash, script-hash, SegWit/Taproot, and MWEB extension addresses). The project changes the **network prefix bytes and human-readable parts (HRPs)** so that Zorkcoin addresses are distinct from Litecoin addresses; those prefix values are defined in the Zorkcoin source tree on GitHub. @todo: extract and list the exact prefix hex/decimal values from `chainparams` in the ZorkNetwork/zorkcoin repo.

---

### Primary address types (recommended)
#### Taproot addresses (P2TR)
- **Format**: Bech32m human-readable part + witness program (v1, 32-byte key) — modern native Taproot format. **Expected default for Zorkcoin** receive addresses because of privacy, efficiency, and future-proofing. Bech32m encoding and semantics follow BIP-350/BIP-341 conventions used by Bitcoin/Litecoin for Taproot. @todo: confirm Zorkcoin HRP for Taproot (e.g., `zork1` or similar) in source code.

#### MWEB addresses
- **Format**: Bech32-style encoding of a serialized pair of MWEB pubkeys (scan and spend); used for MimbleWimble Extension Block (MWEB) peg-ins and confidential outputs.
- **Semantics**: MWEB addresses are *not* standard Bitcoin scriptPubKey outputs; they encode the MWEB account keys and are used to peg funds into the MWEB extension block. Wallets must implement MWEB-specific RPCs and UTXO tracking to manage these outputs.
- **Recommendation**: Use MWEB addresses for privacy-sensitive transfers and for interacting with the MWEB UTXO set; treat them as first-class address types on Zorkcoin. @todo: confirm Zorkcoin MWEB HRP and exact serialization details in Zorkcoin code or mwebd-compatible libraries.

---

### Compatibility and older address styles
#### SegWit Bech32 P2WPKH and P2WSH
- **Format**: Bech32 HRP + witness program (v0). Supported for compatibility with existing wallets and hardware devices; lower fees than legacy addresses.
- **Notes**: Zorkcoin will accept these but users should prefer Taproot for new addresses.

#### P2SH and P2PKH (Base58Check legacy)
- **Format**: Base58Check with a single version byte prefix for P2PKH and P2SH. These remain supported for backward compatibility but are **not recommended** for new usage due to larger size and weaker feature set.
- **Security**: Legacy addresses reveal less of the script type and are widely supported, but they lack Taproot privacy and MWEB confidentiality.

---

### Developer and wallet implementer notes
- **Address validation**: Wallets must validate both the encoding (Base58Check, Bech32, Bech32m) and the network-specific prefix/HRP to avoid cross-chain mistakes. @todo: add a short code snippet showing how to check Zorkcoin HRP and version bytes once prefixes are confirmed.
- **MWEB integration**: Implementations should follow the mwebd conventions for deriving scan/spend keys (BIP32 derivation examples) and for address serialization used by existing MWEB tooling.
- **Testing**: Use testnet prefixes and HRPs (if provided in the repo) to avoid sending real funds while integrating.

