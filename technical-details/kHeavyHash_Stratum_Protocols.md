---
layout: default
title: kHeavyHash Stratum Protocols
nav_order: 4
---

# kHeavyHash Stratum Protocols

Here’s a detailed breakdown of how to **feed work to kHeavyHash miners** and **submit results**, based on Stratum-like protocols used by most ASIC devices:

Here’s a deep dive backed by actual vendor documentation showing how **kHeavyHash ASIC miners** implement the Stratum JSON‑RPC interface—not just assumptions:

```{note}
This is a work in progress.  This note will be removed after a proofreading.  Feel free to contribute by asking questions where further clarification is required, or by providing constructive feedback.
```

---

## 🔍 Official JSON‑RPC/Stratum Details from IceRiver & Bitmain

### IceRiver KS5L & KS3 (IceRiverMiner-v1.1)

The **KS5L user manual** outlines standard Stratum methods:

1. `mining.subscribe`
2. `mining.set_difficulty`
3. `mining.set_extranonce`

   > *nonce-prefix is the first part of the block header nonce (in hex)… nonce2 length = 8 − len(nonce-prefix)*
4. `mining.authorize`
5. `mining.notify`
6. `mining.submit`
   {CITATION\_START}cite{CITATION\_DELIMITER}turn0search16{CITATION\_STOP}

The manual even details JSON examples:

```json
{ "id":1, "method":"mining.subscribe", "params":[
  "IceRiverMiner-v1.1","EthereumStratum/1.0.0"] }

{ "id":2, "method":"mining.set_difficulty", "params":[4096] }

{ "id":3, "method":"mining.set_extranonce", "params":["0001"] }

// job notification
{ "id":1, "method":"mining.notify", "params":[
  "34","2461684d…f7838b4bf75ea7c9222a" ] }

{ "id":140, "method":"mining.submit", "params":[
  "kaspa:…worker","34","00011f7a5745732a"] }
```

{CITATION\_START}cite{CITATION\_DELIMITER}turn0search16{CITATION\_STOP}

These confirm exactly how parameters like `nonce-prefix` (extranonce) and 8‑byte nonce composition are structured.

---

### Bitmain KS5 Pro Manual

The **Bitmain KS5 Pro user guide** (via Manuals+) confirms support for KHeavyHash and standard Stratum, including pool configuration:

* Using JSON-RPC methods like `mining.subscribe`, `authorize`, and pool URLs
  {CITATION\_START}cite{CITATION\_DELIMITER}turn0search4{CITATION\_STOP}

Although it doesn’t list JSON payloads, it's consistent with the IceRiver layout, confirming Bitmain ASICs use the same interface.

---

## 📖 Stratum Fundamentals

These miners follow **Stratum V1 (JSON‑RPC 2.0)**, documented by Stratum protocol guides:

* Uses newline-delimited JSON (per Stratum spec)
* Supports methods: `mining.subscribe`, `get_extranonce.subscribe`, `mining.authorize`, `mining.submit`, and server push `mining.notify`, etc.
  {CITATION\_START}cite{CITATION\_DELIMITER}turn0search5{CITATION\_DELIMITER}turn0search17{CITATION\_STOP}


---


## ⚙️ Stratum Protocol for kHeavyHash Miners

### Protocol Overview

* kHeavyHash miners (ASICs like Bitmain, IceRiver, etc.) typically connect to **pools via Stratum (V1/V2 variants)**, using a JSON-RPC interface over TCP or SSL ([Manuals+][1], [cdn.webshopapp.com][2]).
* The protocol supports common messages: `mining.subscribe`, `set_difficulty`, `set_extranonce`, `mining.authorize`, `mining.notify`, and `mining.submit` ([Manuals+][1]).

---

### 🔌 Subscription Flow

**Client → Pool**

```json
{ "id":1,
  "method":"mining.subscribe",
  "params":["MinerNAME-v1.0","EthereumStratum/1.0.0"] }
```

**Pool → Client**

```json
{ "id":1,
  "result":[true,"EthereumStratum/1.0.0"], "error":null }
```

* This registers the miner; **protocol version aligns with other Stratum-based miners** ([Manuals+][1]).

---

### ⚙️ Difficulty & Extra Nonce

* Pool adjusts partial share difficulty:

  ```json
  { "id":2,
    "method":"mining.set_difficulty",
    "params":[4096] }
  ```
* Extra-nonce is used to provide additional unique bits:

  ```json
  { "id":3,
    "method":"mining.set_extranonce",
    "params":["0001"] }
  ```
* The miner uses this in constructing its 8-byte nonce ([Manuals+][1]).

---

### 🧩 Authorize

* Miner authenticates using credentials (e.g., `username`: `kaspa:WALLET.WORKER`).

  ```json
  {"id":3, "method":"mining.authorize",
   "params": ["kaspa:<address>.<worker>",""] }
  ```

---

### 🔔 Job Notification (`mining.notify`)

When new blockwork is available:

```json
{"id":1,
 "method":"mining.notify",
 "params":[
   "jobId",
   "header_hash"
 ]}
```

* `header_hash` is the 32-byte field miners must use as input to kHeavyHash.

---

### 🔚 Submit

Once a candidate hash meets criteria:

```json
{"id":140,
 "method":"mining.submit",
 "params":[
   "kaspa:<address>.<worker>",
   "jobId",
   "nonce" ]}
```

**Results:**

* Accepted share:

  ```json
  {"id":140,"result":true,"error":null}
  ```
* Rejected due to low diff:

  ```json
  {"id":140,"result":null,"error":[30,"Low difficulty share",null]}
  ```

---

## 🧩 Multi-part Nonce Structure

* Kaspa's nonce is **8 bytes** total.
* A prefix (set via `extranonce`) and suffix (miner-chosen nonce) together fill 8 bytes ([Manuals+][1], [NiceHash][3], [Coin Guides][4]).
* Example: Extranonce “0001” (2 bytes), suffix is 6 bytes random.

---

## 🔧 ASIC Integration Summary

1. **Connect**: `mining.subscribe`
2. **Authenticate**: `mining.authorize`
3. **Receive** jobs: `mining.notify`
4. **Compute** hash (kHeavyHash(header||nonce))
5. **Submit** solution: `mining.submit`
6. **Pool validates** and responds

This cycle continues as new jobs come in.

---

## 🔄 Stratum Adapters for Solo Mining

* To use **Stratum-capable ASICs** with a **solo Kaspa node**, miners often use adapters:

  * **kaspa-stratum-bridge** (Go) by onemorebsmith/rdugan ([Manuals+][1], [NiceHash][3], [2miners.com][5], [Kaspa Wiki][6])
  * **KStratum** (JavaScript) by KaffinPX ([Kaspa Wiki][7])
* These proxies:

  * Call the Kaspa node API to generate block templates.
  * Translate them into Stratum `mining.notify`.
  * Relay `mining.submit` from miner back to the Kaspa node.

---

### ✅ Summary

* kHeavyHash ASICs use **almost standard Ethereum-flavored Stratum**. The only key differences are:

  * 8-byte nonce with prefix handling.
  * `header_hash` transporting kHeavyHash-compatible input.
* Open-source Stratum proxies allow **solo** mining via Stratum interface.
* This matches **most community and vendor documentation** ([Manuals+][1], [NiceHash][3], [Kaspa Wiki][6]).

---

Would you like:

* A sample **Stratum job JSON** including nonce format?
* Help configuring a proxy like `kaspa-stratum-bridge`?
* A demo of how to parse `mining.notify` or integrate this into firmware/software?

[1]: https://manuals.plus/m/f20a9136b47412a5c58d207d7b7576e0cba08ff05cdbe492dce44e974c371d3e?utm_source=chatgpt.com "Kaspa Common Stratum Protocol Kaspa Common ... - Manuals.plus"
[2]: https://cdn.webshopapp.com/shops/353164/files/464331150/ks5l-operation-manual-en.pdf?utm_source=chatgpt.com "[PDF] KS5L User Manual"
[3]: https://www.nicehash.com/blog/post/kaspa-kheavyhash-is-now-supported-on-nicehash?utm_source=chatgpt.com "Kaspa (kheavyhash) is now supported on NiceHash!"
[4]: https://coinguides.org/kaspa-mining/?utm_source=chatgpt.com "How to mine Kaspa - GPU mining KAS (kHeavyHash) - Coin Guides"
[5]: https://2miners.com/blog/nicehash-and-asic-compatible-stratum-protocols-on-2miners/?utm_source=chatgpt.com "NiceHash and ASIC-compatible Stratum Protocols on 2Miners"
[6]: https://wiki.kaspa.org/node?utm_source=chatgpt.com "Node - Kaspa WIKI"
[7]: https://wiki.kaspa.org/en/software-ecosystem?utm_source=chatgpt.com "Software Ecosystem - Kaspa WIKI"



