---
layout: default
title: Legacy Soft Forks Enabled Post-Genesis
nav_order: 8
lang: en
published: false
nav_exclude: true
search_exclude: true
hide: true
---

# Legacy Soft Forks Enabled Post-Genesis

Some Bitcoin-era upgrades behave as genesis baseline rules on Zork Network—see [Legacy Soft Forks Inherited at Genesis](Legacy_Soft_Forks_Inherited_at_Genesis.md). Litecoin later enabled additional soft forks on its own timetable. Because Zork Network follows that codebase lineage, block explorers and node tooling typically list those deployments separately, with heights and statuses that indicate when the rules reached active status, rather than behaving like inherited-from-genesis milestones.

This page reserves space for that second generation of forks. The sections below begin as outlines and checklists until the narratives are drafted.

## Taproot

- [ ] Summarize Taproot consensus changes in plain language (witness version 1, Schnorr signatures, tweaked keys, Merkelized Alternative Script Trees), and distinguish consensus from wallet-visible address conventions.
- [ ] Map explorer labels and prominent BIPs readers will encounter alongside deployment history on upstream Litecoin-class chains without reproducing entire specifications here.
- [ ] Describe how this matters practically for wallets and auditors on Zork Network once parameters and activation context are spelled out elsewhere in the docs.

## MimbleWimble Extension Blocks (MWEB)

- [ ] Explain MWEB as an optional confidentiality layer (private transfers alongside transparent chain flows) pegged into the main ledger, consistent with Litecoin-era design.
- [ ] Cover peg-in, peg-out, and extension-block roles at overview level plus how explorers usually surface balances versus extension movement.
- [ ] Align privacy wording with product guidance (optional privacy, plain-language “private transfers,” no anonymity hype).
- [ ] Tie documentation to consensus or policy flags Zork inherits from Litecoin-derived clients once authoritative parameters are summarized on this site.
