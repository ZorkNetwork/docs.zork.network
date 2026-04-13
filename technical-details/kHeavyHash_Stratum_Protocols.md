---
layout: default
title: kHeavyHash Stratum Protocols
nav_order: 2
---

# kHeavyHash Stratum Protocols

This document provides a complete specification of the Stratum protocol as implemented by kHeavyHash-compatible ASIC miners. It covers both the **miner (client)** and **pool/portal/bridge/proxy (server)** perspectives, enabling developers to implement either mining software or Stratum server software. The protocol is used for mining cryptocurrencies that employ the kHeavyHash proof-of-work algorithm, including Kaspa (KAS) and Zorkcoin (ZORK).

---

## Protocol Overview

kHeavyHash ASIC miners (manufactured by Bitmain, IceRiver, and others) communicate with mining pools using **Stratum V1**, a JSON-RPC 2.0 based protocol over TCP or SSL/TLS. The protocol follows an Ethereum-flavored Stratum specification with kHeavyHash-specific adaptations for nonce handling and block header structure.

### Key Characteristics

- **Protocol:** Stratum V1 (JSON-RPC 2.0)
- **Transport:** TCP or SSL/TLS (persistent connection)
- **Message Format:** Newline-delimited JSON
- **Encoding:** UTF-8
- **Nonce Size:** 8 bytes total (when extranonce is used: server-contributed prefix + miner-contributed portion)
- **Work input to kHeavyHash:** 80 bytes (PrePowHash + timestamp + padding + nonce). Work is delivered to miners via one of two mechanisms: **Standard Format** (job fields sent separately; miner builds the 80-byte input) or **BigJob Format** (full 80-byte work header sent as a single hex string). See [mining.notify](#miningnotify).

### Protocol Version

- **JSON-RPC Version:** 2.0
- **Stratum Version:** EthereumStratum/1.0.0 (standard format)
- **Extranonce Support:** Optional (for multi-client scenarios)

### Clients and Compatibility

The same Stratum interface defined here is used by both hardware and software miners. Anything
compatible with kHeavyHash Stratum mining will work with this protocol specification.

**Hardware (ASIC):**
- **IceRiver** — kHeavyHash ASIC miners
- **Bitmain** — kHeavyHash ASIC miners (Antminer KS series)
- Other kHeavyHash ASIC manufacturers

**Software (CPU/GPU):**
- **BzMiner** — GPU miner (Nvidia, AMD; Kaspa stratum support)
- **lolminer** — GPU miner (stratum-bridge compatible)
- **SRBMiner** — Multi-algorithm miner with Kaspa/kHeavyHash support
- **TeamRedMiner** — AMD GPU miner (bridge-compatible in some configurations)
- Other kHeavyHash-capable CPU/GPU mining software that speak Stratum

**Work delivery:** Servers may send work in one of two ways. **Standard Format** sends job ID, PrePowHash (as 4× uint64), and timestamp; the miner constructs the 80-byte work input. **BigJob Format** sends job ID and the full 80-byte work header as a hex string (used by some ASICs and by BzMiner/IceRiverMiner-style user agents). Server chooses format per client (e.g. by user agent); see [mining.notify](#miningnotify). New miners should utilize
the standard format since it is more efficient.

---

## Connection Establishment

### TCP Connection

1. Miner establishes TCP connection to server on the host and port provided by the pool or bridge (e.g. in pool documentation or configuration). The Stratum protocol does not specify or require a particular port, each server chooses its own.
2. No initial handshake required — connection is immediately ready for JSON-RPC messages.
3. Connection remains open until:
   - Miner disconnects
   - Server disconnects (error, timeout, or shutdown)
   - Network error occurs

### Connection Requirements

- **TCP_NODELAY:** Recommended for lower latency
- **Keep-Alive:** Optional but recommended
- **Timeout:** Server implementations often use read deadlines (e.g. 5 seconds) for timeout handling
- **Message size:** Stratum messages are typically under 1 KB. The protocol does not define a maximum message size; implementations may impose a limit as a safety measure
- **SSL/TLS:** Optional but recommended for production pools

---

## Message Format

All messages follow JSON-RPC 2.0 format with newline-delimited transmission (NDJSON/JSONL). Each complete JSON object is terminated by a newline character (`\n`), forming one protocol message.

### Request Format (Miner → Server)

```json
{
  "id": <number|string|null>,
  "jsonrpc": "2.0",
  "method": "<method_name>",
  "params": [<param1>, <param2>, ...]
}
```

### Response Format (Server → Miner)

```json
{
  "id": <number|string|null>,
  "jsonrpc": "2.0",
  "result": <result_value>,
  "error": null
}
```

### Error Response Format (Server → Miner)

```json
{
  "id": <number|string|null>,
  "jsonrpc": "2.0",
  "result": null,
  "error": [<error_code>, "<error_message>", <error_data>]
}
```

### Notification Format (Server → Miner)

```json
{
  "jsonrpc": "2.0",
  "method": "<method_name>",
  "params": [<param1>, <param2>, ...]
}
```

**Note:** Notifications from server to client omit the `"id"` field to indicate they are not responses to requests.

### Message Delimiting

- Messages use newline-delimited JSON (NDJSON/JSONL format): each complete JSON object is terminated by a newline character (`\n`)
- Each complete JSON object forms one protocol message
- In practice, Stratum implementations typically send compact JSON (minimal whitespace, often on a single line) for simplicity and efficiency
- You cannot split a single JSON object across multiple protocol messages (e.g., opening brace in one message, closing brace in another)
- Null bytes (`\x00`) in messages should be stripped by server implementations
- The Stratum protocol does not define a maximum message size; implementations may impose one

### Miner vs server message roles

The **miner** only sends **requests** (`mining.subscribe`, `mining.authorize`, `mining.submit`). The miner never sends a "response" to the server. The **server** sends **responses** (with `result` or `error`) to those requests, and also sends **notifications** (`mining.set_difficulty`, `mining.notify`, `set_extranonce` / `mining.set_extranonce`) which do not have a response.

### Message Ordering

- Requests and responses maintain request ID correlation
- Notifications can arrive at any time
- Server may send multiple notifications before a response
- Messages should be processed in order received

---

## Protocol Flow

The complete connection and mining cycle follows this sequence:

### Phase 1: Connection and Subscription

```
Miner                          Server
  |                               |
  |--- TCP Connect -------------->|
  |                               |
  |--- mining.subscribe --------->|
  |                               |
  |<-- Response ------------------|
  |   [subscription details]      |
  |                               |
  |--- mining.authorize --------->|
  |   [username, password]        |
  |                               |
  |<-- Response ------------------|
  |   [true/false]                |
  |                               |
  |<-- mining.set_extranonce -----| (if extranonce enabled)
  |   [extranonce, size]          |
  |                               |
  |<-- mining.set_difficulty -----|
  |   [difficulty_value]          |
```

### Phase 2: Mining

```
Server                          Miner
  |                               |
  |<-- mining.set_difficulty -----|
  |   [difficulty_value]          |
  |                               |
  |<-- mining.notify -------------|
  |   [job_id, work_data, ...]    |
  |                               |
  |                               | [Miner mines...]
  |                               |
  |<-- mining.submit -------------|
  |   [worker, job_id, nonce]     |
  |                               |
  |--- Response ----------------->|
  |   [true/false or error]       |
```

### Complete Flow Diagram

```
┌─────────┐                                    ┌─────────┐
│  Miner  │                                    │ Server  │
└────┬────┘                                    └────┬────┘
     │                                              │
     │ 1. TCP Connect                               │
     │─────────────────────────────────────────────>│
     │                                              │
     │ 2. mining.subscribe                          │
     │─────────────────────────────────────────────>│
     │                                              │
     │ 3. Response: [true, "EthereumStratum/1.0.0"] │
     │<─────────────────────────────────────────────│
     │                                              │
     │ 4. mining.authorize                          │
     │─────────────────────────────────────────────>│
     │                                              │
     │ 5. Response: true                            │
     │<─────────────────────────────────────────────│
     │                                              │
     │ 6. mining.set_extranonce (if enabled)        │
     │<─────────────────────────────────────────────│
     │                                              │
     │ 7. mining.set_difficulty                     │
     │<─────────────────────────────────────────────│
     │                                              │
     │ 8. mining.notify                             │
     │<─────────────────────────────────────────────│
     │                                              │
     │ [Miner mines work...]                        │
     │                                              │
     │ 9. mining.submit                             │
     │─────────────────────────────────────────────>│
     │                                              │
     │ 10. Response: true                           │
     │<─────────────────────────────────────────────│
     │                                              │
     │ [Repeat steps 7-10 as new blocks arrive]     │
     │                                              │
```

---

## Message Reference

### mining.subscribe

**Direction:** Miner → Server  
**Type:** Request  
**Purpose:** Subscribe to mining notifications and identify miner software

**Request:**
```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "mining.subscribe",
  "params": ["<user_agent>", "<protocol_version>"]
}
```

**Parameters:**
- `params[0]` (string, optional): User agent string identifying the miner software
  - Example: `"BzMiner/1.0.0"`, `"IceRiverMiner-v1.1"`, `"BitmainMiner/1.0"`, `"VirtualMiner/1.0"`
- `params[1]` (string, optional): Protocol version string, typically `"EthereumStratum/1.0.0"`

**Response (Standard Format):**
```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": [true, "EthereumStratum/1.0.0"]
}
```

**Response (Bitmain Format - if extranonce enabled and Bitmain client detected):**
```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": [null, "<extranonce_hex>", <miner_nonce_length>]
}
```

**Response Parameters:**
- Standard format:
  - `result[0]`: `true` (boolean)
  - `result[1]`: `"EthereumStratum/1.0.0"` (protocol version string)
- Bitmain format (when extranonce enabled):
  - `result[0]`: `null`
  - `result[1]`: Extranonce as hexadecimal string (e.g. `"00000001"`)
  - `result[2]`: Length in bytes of the miner’s nonce portion (same meaning as second parameter of `mining.set_extranonce` / `set_extranonce`)

**Server Behavior:**
- Stores user agent in client context
- Detects Bitmain clients via regex: `.*(GodMiner).*` (Bitmain-specific)
- Assigns extranonce if extranonce is enabled
- Response format depends on client type and extranonce configuration
- After sending response, server should send `mining.set_difficulty` and `mining.set_extranonce` (if enabled)

**Miner Behavior:**
- Should store the subscription result for protocol validation
- The `id` field should be unique per request and used to match responses
- After receiving the response, wait for `mining.set_difficulty` and `mining.set_extranonce` before proceeding

---

### mining.authorize

**Direction:** Miner → Server  
**Type:** Request  
**Purpose:** Authenticate miner.

**Request:**
```json
{
  "id": 2,
  "jsonrpc": "2.0",
  "method": "mining.authorize",
  "params": ["<username>", "<password>"]
}
```

**Parameters:**
- `params[0]` (string, required): Username. Interpretation is server-defined. Many servers use wallet address and optional worker name as the username (e.g. `"<address>"` or `"<address>.<worker>"`; some pools use a prefix such as `"kaspa:<address>.<worker>"`).
- `params[1]` (string, optional): Password. Often empty string `""`.

**Response (Success):**
```json
{
  "id": 2,
  "jsonrpc": "2.0",
  "result": true
}
```

**Response (Failure):**
```json
{
  "id": 2,
  "jsonrpc": "2.0",
  "result": false
}
```

**Server Behavior:**
- Validates credentials according to pool or bridge policy (e.g. username/password or address format)
- Triggers work sending after successful authorization
- Sends extranonce notification if extranonce is enabled
- Sends initial `mining.set_difficulty` after authorization
- Sends `mining.notify` if cached block template exists
- May disconnect client if authorization fails or after a timeout

**Miner Behavior:**
- Authorization must succeed before the miner can receive jobs or submit shares
- Username and password format are defined by the server; check pool or bridge documentation

---

### mining.set_difficulty

**Direction:** Server → Miner  
**Type:** Notification (no ID)  
**Purpose:** Set or update mining difficulty

**Notification:**
```json
{
  "jsonrpc": "2.0",
  "method": "mining.set_difficulty",
  "params": [<difficulty>]
}
```

**Parameters:**
- `params[0]` (number, required): Difficulty value as floating-point number
  - Example: `1.0`, `2.5`, `0.5`, `16.0`, `4096.0`
  - Minimum: `0.0078125` (2^-7)
  - No maximum; range is implementation-defined (e.g. up to 10000 or more, per pool policy)
  - Some implementations use integer values (e.g. `4096`); floating-point is common

**Server Behavior:**
- Sent before first `mining.notify` (initial difficulty)
- Sent immediately after successful `mining.authorize`
- Sent whenever difficulty changes (e.g. when server uses VarDiff; see [Variable Difficulty (VarDiff)](#variable-difficulty-vardiff))
- Difficulty applies to all subsequent shares until changed
- Miner must update difficulty immediately upon receipt

**Miner Behavior:**
- Must accept and apply difficulty immediately
- Difficulty applies to all shares submitted after this notification
- Should not submit shares below the new difficulty
- Old shares may still be submitted if they meet the new difficulty

**Timing:**
- Always sent before `mining.notify` for new jobs
- May be sent between jobs (e.g. VarDiff adjustment)
- Sent immediately after authorization (initial difficulty)

---

### mining.notify

**Direction:** Server → Miner  
**Type:** Notification (no ID)  
**Purpose:** Send new mining job to miner

The server may use one of two formats depending on the miner type:

#### Standard Format (Most Miners)

```json
{
  "jsonrpc": "2.0",
  "method": "mining.notify",
  "params": [
    "<job_id>",
    [<uint64_0>, <uint64_1>, <uint64_2>, <uint64_3>],
    <timestamp>
  ]
}
```

**Parameters:**
- `params[0]` (string): Job ID (unique identifier for this job)
  - Example: `"12345"`
  - Used in `mining.submit` to reference this job
- `params[1]` (array): PrePowHash as 4 uint64 values. **Byte order: little-endian.** Each array element is one 64-bit word; together they form the 32-byte PrePowHash in little-endian order (first element = bytes 0–7, etc.).
  - Example: `[0x0123456789abcdef, 0xfedcba9876543210, 0x1234567890abcdef, 0xfedcba0987654321]`
- `params[2]` (number): Unix timestamp as uint64. **Byte order:** when encoded into the 80-byte work header, the timestamp is 8 bytes, **little-endian**.
  - Example: `1699123456`
  - Used in work header construction

#### BigJob Format (BzMiner/IceRiverMiner)

```json
{
  "jsonrpc": "2.0",
  "method": "mining.notify",
  "params": [
    "<job_id>",
    "<work_header_hex>"
  ]
}
```

**Parameters:**
- `params[0]` (string): Job ID (same as standard format)
- `params[1]` (string): Complete 80-byte work header as hexadecimal string (160 hex characters). **Byte order:** multi-byte fields in the decoded 80-byte buffer are **little-endian** (PrePowHash bytes 0–31, timestamp bytes 32–39, padding 40–71, nonce bytes 72–79; timestamp and nonce are uint64 little-endian).
  - Contains: PrePowHash (32 bytes) + Timestamp (8 bytes, little-endian uint64) + Padding (32 bytes, zeros) + Nonce (8 bytes, little-endian uint64, initially zero)

**Format Detection:**
- Server detects format based on user agent regex: `.*(BzMiner|IceRiverMiner).*`
- Format is determined once per client and stored in client state
- All subsequent jobs use the same format for that client

**Work Header Structure (80 bytes):**
```
Bytes 0-31:   PrePowHash (32 bytes)
Bytes 32-39:  Timestamp (8 bytes, little-endian uint64)
Bytes 40-71:  Padding (32 bytes, all zeros)
Bytes 72-79:  Nonce (8 bytes, little-endian uint64, initially zero)
```

**Server Behavior:**
- Sent whenever new block template is available
- Job ID increments for each new job
- Old jobs remain valid for up to 8 blocks (work window)
- Sent after `mining.set_difficulty` (if difficulty changed)
- When a new job is sent, previous jobs with different job IDs become invalid

**Miner Behavior:**
- Must construct work header from job data (Standard Format) or use directly (BigJob Format)
- Must increment nonce to find valid shares
- May update timestamp if needed (within valid range)
- Should submit shares using correct job ID
- Should discard stale jobs (> 8 blocks old)

**Job Lifecycle:**
- Jobs are valid until:
  - New block is found (network advances)
  - Job becomes stale (> 8 blocks old)
  - Server explicitly invalidates job
- Miner should track multiple jobs and submit to correct one

---

### mining.submit

**Direction:** Miner → Server  
**Type:** Request  
**Purpose:** Submit a share (valid work result)

**Request:**
```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "method": "mining.submit",
  "params": [
    "<worker_name>",
    "<job_id>",
    "<nonce_hex>"
  ]
}
```

**Parameters:**
- `params[0]` (string): Username or worker identifier (optional; may be empty string `""`). Often the same value as used in `mining.authorize`; used by server for statistics and attribution.
- `params[1]` (string): Job ID from `mining.notify`
  - Must match a valid, non-stale job
  - Example: `"12345"`
- `params[2]` (string): Nonce as hexadecimal string
  - If extranonce is **not enabled**: Must be exactly 16 hex characters (8 bytes), representing the complete nonce
  - If extranonce **is enabled**: May be either:
    - The complete 8-byte nonce (16 hex characters), OR
    - Only the miner-contributed portion (typically 6-8 bytes, 12-16 hex characters)
  - Example (full nonce): `"1234567890abcdef"`
  - Example (miner portion only): `"567890abcdef"` (when server contributes 2 bytes)
  - May include `"0x"` prefix (automatically stripped by server)
  - Server implementations should handle both formats when extranonce is enabled

**Response (Success):**
```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "result": true
}
```

**Response (Error):**
```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "result": null,
  "error": [<error_code>, "<error_message>", null]
}
```

**Server Behavior:**
1. Validates job ID exists and is not stale
2. Constructs full nonce:
   - If extranonce is enabled and miner sent only miner-contributed portion: prepend server-contributed portion (extranonce) to form complete 8-byte nonce
   - If miner sent complete nonce: use as-is (server may validate that server-contributed portion matches if extranonce is enabled)
   - If extranonce is not enabled: use nonce as-is
3. Validates share meets stratum difficulty
4. Checks for duplicate shares
5. If share meets network difficulty, submits block to network
6. Records statistics (accepted/rejected shares)

**Share Validation:**
- Share must meet stratum difficulty (set via `mining.set_difficulty`)
- Share must reference valid, non-stale job
- Share must not be duplicate
- If share meets network difficulty, block is submitted

**Error Codes:**
- `20`: Unknown problem (invalid share, bad PoW)
- `21`: Job not found (stale job)
- `22`: Duplicate share
- `23`: Invalid difficulty (share below stratum difficulty)

**Special Handling:**
- Some ASICs (IceRiver, Bitmain) submit wrong job IDs
- Server implementations may search job history if share is invalid
- If correct job found, share is accepted with warning logged
- Maximum search: typically 32 jobs (one full job window)

**Miner Behavior:**
- If extranonce is **not enabled**: Submit the complete 8-byte nonce (16 hex characters)
- If extranonce **is enabled**: May submit either:
  - The complete 8-byte nonce (16 hex characters), OR
  - Only the miner-contributed portion (length specified in `mining.set_extranonce` second parameter)
- Submit only shares that meet the difficulty target specified by `mining.set_difficulty`
- The miner should track submitted nonces to avoid duplicates
- Job IDs expire when new jobs are received; do not submit shares for expired jobs

---

### mining.extranonce.subscribe

**Direction:** Miner → Server  
**Type:** Request  
**Purpose:** Request extranonce notifications (optional)

**Request:**
```json
{
  "id": 4,
  "jsonrpc": "2.0",
  "method": "mining.extranonce.subscribe",
  "params": []
}
```

**Response:**
```json
{
  "id": 4,
  "jsonrpc": "2.0",
  "result": true
}
```

**Server Behavior:**
- Always returns `true`
- Extranonce is sent automatically after authorization if enabled
- This method is for protocol compatibility

---

### mining.set_extranonce / set_extranonce

**Direction:** Server → Miner  
**Type:** Notification (no ID)  
**Purpose:** Notify miner of extranonce (if extranonce enabled)

**Notification (Standard Format):**
```json
{
  "jsonrpc": "2.0",
  "method": "set_extranonce",
  "params": ["<extranonce_hex>", <miner_nonce_length>]
}
```

**Notification (Bitmain Format):**
```json
{
  "jsonrpc": "2.0",
  "method": "mining.set_extranonce",
  "params": ["<extranonce_hex>", <miner_nonce_length>]
}
```

**Parameters:**
- `params[0]` (string): Extranonce as hexadecimal string (server-assigned prefix of the full 8-byte nonce).
  - Example: `"0001"` = 2 bytes (4 hex chars). The **whole nonce** is always 8 bytes; the remainder is for the miner.
- `params[1]` (number): **Length in bytes of the miner’s nonce portion** (the part the miner chooses). Equals `8 − (length of extranonce in bytes)`. So if `params[0]` is 4 hex characters (2 bytes), the miner has 6 bytes to fill: `params[1]` = 6.

**Example:** Extranonce `"0001"` (2 bytes) → miner nonce length = 6 bytes. Full nonce = 2 + 6 = 8 bytes. In `mining.submit` the miner sends the **nonce** (either the full 8 bytes or just the 6-byte miner portion); if only the 6 bytes are sent, the server prepends the 2-byte extranonce when validating.

**Server Behavior:**
- Sent after successful `mining.authorize` if extranonce is enabled
- Method name depends on client type (Bitmain vs standard)
- Extranonce is unique per client connection
- Full nonce = extranonce concatenated with miner-contributed portion. In `mining.submit` the miner sends the **nonce** (full 8 bytes or only the miner-contributed portion).
- Changing extranonce invalidates all previous jobs; send a new `mining.notify` after changing extranonce

**Miner Behavior:**
- Prepend extranonce to the miner-chosen bytes to form the full 8-byte nonce before hashing
- In `mining.submit`, send the **nonce**: either the complete 8-byte nonce or only the miner-contributed bytes (length = `params[1]`)
- Server prepends extranonce when validating if only the miner portion was sent

---

## Nonce Construction

The kHeavyHash algorithm requires an 8-byte nonce. Miners search over possible nonce values to find one that yields a hash meeting the difficulty target. In the Stratum protocol, the server may assign an **extranonce** (server-contributed portion) so that each connected miner searches a **different nonce space**—avoiding duplicate work and nonce collisions when many miners are connected.

When extranonce is enabled, the 8-byte nonce is built from two concatenated parts:

1. **Server-contributed portion** (protocol name: **extranonce**): Sent by the server in `mining.set_extranonce` (typically 0–4 bytes). The server gives each connection a different value so miners search different ranges.
2. **Miner-contributed portion**: The bytes the miner chooses (remaining bytes to total 8). The miner searches over this part; the protocol does not give this part a separate name—in `mining.submit` the miner sends a **nonce** (either the full 8 bytes or only this portion).

### Nonce Format

The complete 8-byte nonce is formed by **concatenating** the server-contributed portion followed by the miner-contributed portion:

```
Complete Nonce (8 bytes) = [Server-contributed] || [Miner-contributed]
```

Where `||` denotes byte concatenation.

### Byte Layout with Extranonce

When extranonce is enabled, the 8-byte nonce is divided as follows:

**Example with 2-byte server contribution:**
```
Server Bit Mask:   0xFFFF000000000000
Miner Bit Mask:    0x0000FFFFFFFFFFFF

Hex Example:       0x00011f7a5745732a
Server Extranonce: 0x0001
Miner Portion:         0x1f7a5745732a
```

**Example with 4-byte server contribution:**
```
Server Bit Mask:   0xFFFFFFFF00000000
Miner Bit Mask:    0x00000000FFFFFFFF

Hex Example:       0x000102035745732a
Server Extranonce: 0x00010203
Miner Portion:             0x5745732a
```

**When extranonce is not enabled:**
```
Server Bit Mask:   0x0000000000000000
Miner Bit Mask:    0xFFFFFFFFFFFFFFFF

Hex Example:       0x123456789abcdef0
Server Extranonce: 
Miner Portion:     0x123456789abcdef0
```

### Implementation for Miners

1. Receive `mining.set_extranonce` with extranonce hex string (e.g. `"0001"`) and second parameter = length in bytes of the miner’s nonce portion
2. Miner’s portion length = second parameter (e.g. 6 when extranonce is 2 bytes)
3. Generate random or sequential bytes for the miner-contributed portion
4. Concatenate to form complete nonce: `server_bytes || miner_bytes` (always 8 bytes total)
5. Use complete nonce in hash computation
6. In `mining.submit`, may send either:
   - Complete 8-byte nonce, OR
   - Only the miner-contributed portion (server will prepend its portion)

### Implementation for Servers

1. Generate unique server-contributed portion (extranonce) per miner connection (typically 0-4 bytes)
2. Send via `mining.set_extranonce` or `set_extranonce` (depending on client type):
   - First parameter: server-contributed portion as hex string
   - Second parameter: length in bytes of miner-contributed portion (`8 - server_portion_length`)
3. When validating `mining.submit`:
   - If miner sent complete nonce: extract and verify server-contributed portion matches
   - If miner sent only miner-contributed portion: prepend server-contributed portion to form complete 8-byte nonce
4. Use complete 8-byte nonce for share validation

**Important:** The server-contributed portion (extranonce) puts each miner in a different nonce search space, so they do different work and do not collide; this is how the server distributes work across multiple connections.

---

## Work Header Construction

The work header is an 80-byte structure used as input to the kHeavyHash algorithm. It consists of:

```
Bytes 0-31:   PrePowHash (32 bytes)
Bytes 32-39:  Timestamp (8 bytes, little-endian uint64)
Bytes 40-71:  Padding (32 bytes, all zeros)
Bytes 72-79:  Nonce (8 bytes, little-endian uint64)
```

### PrePowHash Calculation

PrePowHash is calculated from the block header with timestamp=0 and nonce=0:

```
Block Header (88 bytes):
- Version: 4 bytes (int32, little-endian)
- PrevBlock: 32 bytes (Hash)
- MerkleRoot: 32 bytes (Hash)
- Timestamp: 8 bytes (uint64, set to 0)
- Bits: 4 bytes (uint32, little-endian)
- Nonce: 8 bytes (uint64, set to 0)

PrePowHash = SHA256(SHA256(Header))
```

**Note:** This is a simplified description. For exact implementation, refer to the specific blockchain's block header structure (Kaspa, Zorkcoin, etc.).

### Work Header from Standard Format

When receiving `mining.notify` in Standard Format:

1. Extract PrePowHash from `params[1]` (4 uint64 values → 32 bytes, little-endian)
2. Extract timestamp from `params[2]` (uint64)
3. Construct 80-byte header:
   - Bytes 0-31: PrePowHash
   - Bytes 32-39: Timestamp (little-endian uint64)
   - Bytes 40-71: Padding (32 zero bytes)
   - Bytes 72-79: Nonce (little-endian uint64, initially zero, then incremented)

### Work Header from BigJob Format

When receiving `mining.notify` in BigJob Format:

1. Decode hex string directly (160 hex chars = 80 bytes)
2. Header is already complete
3. Only the nonce portion (bytes 72-79) needs to be incremented

### For Pool Servers

Pool servers typically:
1. Receive block templates from network nodes or generate them locally
2. Serialize the block header (excluding nonce field)
3. Compute PrePowHash using double SHA-256 (or blockchain-specific method)
4. Send PrePowHash and timestamp to miners via `mining.notify` (Standard Format)
5. Or send complete work header via `mining.notify` (BigJob Format)

### For Solo Mining Bridges

Solo mining bridges (like `kaspa-stratum-bridge`) must:
1. Connect to a blockchain node (e.g., kaspad for Kaspa)
2. Call the node's block template API (typically `GetBlockTemplate` or similar)
3. Extract block header data from the template
4. Compute PrePowHash from block header
5. Send PrePowHash and timestamp to Stratum miners (Standard Format)
6. Or construct complete work header and send (BigJob Format)

---

## Mining Algorithm

### kHeavyHash Overview

kHeavyHash is the proof-of-work algorithm used by cryptocurrencies that employ this Stratum protocol. It operates on an 80-byte work header.

### kHeavyHash Algorithm Steps

1. **Input:** 80-byte work header
2. **Process:**
   - Compute PrePowHash from work header (if not already provided)
   - Apply kHeavyHash algorithm (see [kHeavyHash Technical Overview](kHeavyHash_Technical_Overview.md) for complete specification)
   - The algorithm uses cSHAKE256, xoshiro256++ PRNG, and matrix operations
3. **Output:** 32-byte hash

**Note:** For the complete kHeavyHash algorithm specification, refer to the [kHeavyHash Technical Overview](kHeavyHash_Technical_Overview.md) document.

### Difficulty Checking

1. **Reverse hash bytes** (Bitcoin-style, big-endian comparison):
   ```python
   reversed_hash = reverse_bytes(hash)
   ```

2. **Compare against target**:
   ```python
   target = max_target / difficulty
   pow_value = int.from_bytes(reversed_hash, 'big')
   if pow_value <= target:
       # Valid share!
   ```

3. **Difficulty levels:**
   - **Stratum difficulty:** Minimum difficulty for share acceptance
   - **Network difficulty:** Difficulty required for block submission
   - Share must meet stratum difficulty
   - Block must meet network difficulty

### Nonce Incrementing

- Nonce is 8 bytes (64 bits)
- Miner increments nonce from 0 to 2^64 - 1
- If extranonce enabled:
  - Full nonce = server-contributed portion (extranonce) concatenated with miner-contributed portion
  - Miner searches over the miner-contributed portion
  - Server prepends its extranonce when validating if miner sent only their portion

### Timestamp Updates

- Timestamp can be updated by miner (within valid range)
- Server validates timestamp is reasonable
- Timestamp affects hash result
- Miner may update timestamp to find valid shares

---

## Error Handling

### Error Response Format

All error responses follow this format:
```json
{
  "id": <request_id>,
  "jsonrpc": "2.0",
  "result": null,
  "error": [<error_code>, "<error_message>", <error_data>]
}
```

### Error Codes

| Code | Name | Description | When It Occurs |
|------|------|-------------|----------------|
| 20 | Unknown problem | Invalid share, bad PoW, or other validation failure | Share doesn't meet difficulty or has invalid PoW |
| 21 | Job not found | Stale job or job doesn't exist | Job ID references non-existent or stale job (> 8 blocks old) |
| 22 | Duplicate share | Share was already submitted | Same nonce/job combination already processed |
| 23 | Invalid difficulty | Share below stratum difficulty | Share doesn't meet minimum difficulty requirement |

**Note:** Some implementations use error code 30 for "Low difficulty share" instead of 23. Both are valid; check pool documentation.

### Error Handling Flow

```
Miner submits share
       |
       v
Server validates job ID
       |
       +-- Invalid/Stale --> Error 21 (Job not found)
       |
       v
Server validates share difficulty
       |
       +-- Below stratum diff --> Error 23 (Invalid difficulty)
       |
       v
Server checks for duplicates
       |
       +-- Duplicate --> Error 22 (Duplicate share)
       |
       v
Server validates PoW
       |
       +-- Invalid PoW --> Error 20 (Unknown problem)
       |
       v
Success: Share accepted
```

### Stale Share Detection

- Jobs are considered stale if block height difference > 8 blocks
- Server tracks current tip height
- Shares for stale jobs are rejected with error 21
- Miner should discard stale jobs and wait for new work

### Duplicate Share Detection

- Server tracks submitted nonces per job
- Same nonce + job ID combination is rejected
- Error 22 indicates duplicate
- Miner should ensure nonce uniqueness

### Low Difficulty Share Handling

- Shares below stratum difficulty are rejected
- Error 23 (or 30) indicates invalid difficulty
- Miner should ensure shares meet difficulty before submitting
- Server may search job history for ASICs with wrong job IDs

### Connection Errors

**Network Errors:**
- Connection reset: Miner should reconnect
- Timeout: Miner should reconnect
- Write failure: Server disconnects client

**Protocol Errors:**
- Invalid JSON: Connection may be closed
- Missing required fields: Error response or disconnect
- Malformed parameters: Error response

**Server-Side Errors:**
- Block template fetch failure: Work sending delayed
- Address validation failure: Client disconnected after timeout
- Share validation failure: Error response, connection remains open

### Implementation Recommendations

**For Miners:**
- Implement exponential backoff for connection retries
- Validate all server responses before processing
- Track job IDs and discard expired jobs
- Handle error responses gracefully and log for debugging
- Implement reconnection logic for dropped connections

**For Servers:**
- Validate all client messages for format and content
- Provide clear error messages in error responses
- Implement rate limiting to prevent abuse
- Log all submissions for auditing and debugging
- Handle malformed messages gracefully without crashing

---

## Special Features

### Extranonce Handling

**Purpose:** Put each connected miner in a **different nonce search space** so they are not all searching the same range. That avoids duplicate work and nonce collisions when many miners connect to the same server.

**How it works:**
1. Server assigns a unique extranonce (server-contributed portion) to each client
2. Full nonce = extranonce concatenated with miner-contributed portion (always 8 bytes total)
3. In `mining.submit`, the miner sends the **nonce**—either the full 8-byte nonce or only the miner-contributed portion
4. If the miner sent only their portion, the server prepends the extranonce to form the full nonce before validation

**Configuration:**
- Extranonce size: Configurable (typically 0–4 bytes)
- Max clients: 2^(8×extranonce_size) − 1
- Miner’s nonce portion length: 8 − extranonce_size bytes (this is the value sent as the second parameter in `set_extranonce` / `mining.set_extranonce`)

**Example:**
```
Extranonce: "01" (1 byte = 2 hex chars)
Miner’s portion: 8 − 1 = 7 bytes (14 hex chars)

Miner chooses 7 bytes: "1234567890abcdef"
Full nonce: "01" + "1234567890abcdef" = "011234567890abcdef" (8 bytes)
Miner submits only: "1234567890abcdef"
```

### Variable Difficulty (VarDiff)

**Note:** VarDiff is **not** part of the Stratum protocol. It is a common **server-side feature** that pools and bridges implement on top of Stratum. When a server uses VarDiff, it still uses the standard `mining.set_difficulty` notification; VarDiff only determines when and how the server changes the difficulty value it sends.

**Purpose:** Automatically adjust per-miner difficulty so that share submission rate stays near a target (e.g. one share every N seconds), reducing load and improving feedback for small miners while keeping share counts manageable for large ones.

**How it is typically implemented:**
1. Server monitors share submission rate per client
2. Compares rate to a target (e.g. 20 shares/minute)
3. Adjusts difficulty up or down based on actual rate
4. Sends `mining.set_difficulty` when an adjustment is applied (same Stratum message as for fixed difficulty)

**Typical tuning (implementation-defined):**
- **Windows:** e.g. 1 min, 3 min, 10 min, 30 min, 60 min, 240 min, then continuous
- **Tolerances:** If rate deviates beyond a band (e.g. ±50%), difficulty is updated
- **Formula:** e.g. `new_diff = old_diff * (actual_rate / target_rate)`
- **Clamping:** Difficulty is often clamped (e.g. to powers of 2), with a minimum (e.g. 0.0078125) and an implementation-defined maximum

### BigJob Format

**Purpose:** Support ASICs that require complete work header as hex string

**Detection:**
- Server detects via user agent regex: `.*(BzMiner|IceRiverMiner).*`
- Format is determined once per client
- All subsequent jobs use same format

**Advantages:**
- Simpler for ASIC firmware
- No need to construct header from parts
- Complete header in single parameter

**Disadvantages:**
- Larger message size (160 hex chars vs ~50 chars)
- Less flexible (can't easily update timestamp)

### Job ID Workaround

**Problem:** Some ASICs (IceRiver, Bitmain) submit shares with wrong job IDs

**Solution:** Server searches job history if share is invalid
1. Validate share against submitted job ID
2. If invalid, try previous job IDs (up to 32 jobs back)
3. If correct job found, accept share with warning
4. If no correct job found, reject as invalid

**Limitations:**
- Only searches if share is below stratum difficulty
- Stops if share meets network difficulty (block found)
- Maximum search: 32 jobs (one full job window)

---

## Implementation Notes

### Message Parsing

**JSON Parsing:**
- Use standard JSON parser (UTF-8)
- Handle both number and string IDs
- Strip null bytes before parsing
- Validate required fields
- Include `"jsonrpc": "2.0"` in all messages

**Line Delimiting:**
- Read until newline (`\n`)
- Handle partial messages (buffer until complete)
- The Stratum protocol does not define a maximum message size; implementations may impose one

### Connection Management

**Server Side:**
- One goroutine/thread per client connection
- 5-second read deadline (timeout handling)
- Write lock to prevent concurrent writes
- Graceful disconnect on errors

**Client Side:**
- Persistent connection
- Reconnect on disconnect
- Handle network errors gracefully
- Exponential backoff for reconnects

### State Management

**Server State (per client):**
- Mining state (jobs, difficulty, etc.)
- Connection state (connected/disconnected)
- Statistics (shares, blocks, etc.)
- Address validation cache
- User agent and format type (Standard/BigJob)
- Extranonce assignment

**Client State:**
- Current job ID
- Current difficulty
- Extranonce (if enabled)
- Connection status
- Work header cache

### Job Management

**Job Storage:**
- Circular buffer (typically 32 jobs)
- Job ID modulo buffer size for indexing
- Old jobs automatically overwritten
- Stale detection: height difference > 8

**Job Lifecycle:**
1. New block template arrives
2. Server creates new job (increments job ID)
3. Server sends `mining.notify` to all clients
4. Clients mine on job
5. Job remains valid until:
   - New block found (network advances)
   - Job becomes stale (> 8 blocks old)
   - Server explicitly invalidates

### Share Validation

**Validation Steps:**
1. Parse and validate request format
2. Check job ID exists and is not stale
3. Construct full nonce (extranonce concatenated with miner-contributed portion, if extranonce enabled)
4. Construct work header (PrePowHash + timestamp + padding + nonce)
5. Calculate block hash using kHeavyHash
6. Check share meets stratum difficulty
7. Check for duplicates
8. If meets network difficulty, submit block
9. Record statistics

**Hash Calculation:**
```python
# Construct work header
header = prePowHash + timestamp + padding + nonce  # 80 bytes

# Apply kHeavyHash
hash = kHeavyHash(header)

# Reverse bytes for comparison
reversed_hash = reverse_bytes(hash)

# Compare against target
target = max_target / difficulty
if int.from_bytes(reversed_hash, 'big') <= target:
    # Valid share
```

### Error Recovery

**Network Errors:**
- Reconnect automatically
- Exponential backoff
- Maximum retry limit

**Protocol Errors:**
- Log error details
- Send error response if possible
- Disconnect if unrecoverable

**Server Errors:**
- Continue operating if possible
- Log errors for debugging
- Disconnect clients only if necessary

### Performance Considerations

**Server:**
- Use connection pooling
- Batch operations where possible
- Cache address validations
- Efficient job storage (circular buffer)

**Client:**
- Minimize message parsing overhead
- Cache work header construction
- Efficient nonce incrementing
- Batch share submissions (if supported)

### Security Considerations

**Credential validation:**
- Validate username/password per pool or bridge policy (e.g. address format, worker naming)
- Cache validation results where appropriate
- Disconnect or reject invalid credentials per policy

**Share Validation:**
- Verify PoW is correct
- Check difficulty requirements
- Prevent duplicate submissions
- Rate limit if necessary

**Connection Security:**
- Consider TLS for production
- Validate message sizes
- Timeout handling
- Resource limits per connection

---

## Complete Example Session

### Connection and Setup

```
1. TCP Connect
   Miner → Server: [TCP connection established]

2. Subscribe
   Miner → Server:
   {
     "id": 1,
     "jsonrpc": "2.0",
     "method": "mining.subscribe",
     "params": ["IceRiverMiner-v1.1", "EthereumStratum/1.0.0"]
   }
   
   Server → Miner:
   {
     "id": 1,
     "jsonrpc": "2.0",
     "result": [true, "EthereumStratum/1.0.0"]
   }

3. Authorize
   Miner → Server:
   {
     "id": 2,
     "jsonrpc": "2.0",
     "method": "mining.authorize",
     "params": ["kaspa:kaspa1qzw6l0x8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8q8.worker1", ""]
   }
   
   Server → Miner:
   {
     "id": 2,
     "jsonrpc": "2.0",
     "result": true
   }

4. Set Extranonce (if enabled)
   Server → Miner:
   {
     "jsonrpc": "2.0",
     "method": "set_extranonce",
     "params": ["0001", 6]
   }

5. Set Difficulty
   Server → Miner:
   {
     "jsonrpc": "2.0",
     "method": "mining.set_difficulty",
     "params": [1.0]
   }

6. Notify (Standard Format)
   Server → Miner:
   {
     "jsonrpc": "2.0",
     "method": "mining.notify",
     "params": [
       "12345",
       [0x0123456789abcdef, 0xfedcba9876543210, 0x1234567890abcdef, 0xfedcba0987654321],
       1699123456
     ]
   }

7. Submit
   Miner → Server:
   {
     "id": 3,
     "jsonrpc": "2.0",
     "method": "mining.submit",
     "params": [
       "worker1",
       "12345",
       "1234567890abcdef"
     ]
   }
   
   Server → Miner:
   {
     "id": 3,
     "jsonrpc": "2.0",
     "result": true
   }
```

### Common Patterns

**Reconnection Pattern:**
```
1. Detect disconnect
2. Wait (exponential backoff)
3. Reconnect TCP
4. Resubscribe
5. Reauthorize
6. Continue mining
```

**Difficulty Update Pattern:**
```
1. Server detects share rate deviation
2. Calculates new difficulty
3. Sends mining.set_difficulty
4. Miner updates difficulty
5. Next mining.notify uses new difficulty
```

**Job Update Pattern:**
```
1. New block template arrives
2. Server increments job ID
3. Server sends mining.set_difficulty (if changed)
4. Server sends mining.notify (new job)
5. Miner switches to new job
6. Old job remains valid (for late shares)
```

---

## Implementation Checklist

### For Miner Developers

- [ ] Implement TCP/SSL connection handling
- [ ] Parse and generate JSON-RPC 2.0 messages (include `"jsonrpc": "2.0"`)
- [ ] Handle `mining.subscribe` request/response
- [ ] Process `mining.set_difficulty` notifications
- [ ] Process `mining.set_extranonce` notifications
- [ ] Implement `mining.authorize` with credential handling
- [ ] Process `mining.notify` to receive jobs (support both Standard and BigJob formats)
- [ ] Construct 80-byte work header from job data
- [ ] Construct 8-byte nonce from extranonce + suffix
- [ ] Implement kHeavyHash computation (or use hardware)
- [ ] Implement difficulty checking (reverse bytes, target comparison)
- [ ] Submit valid shares via `mining.submit`
- [ ] Handle error responses and connection failures
- [ ] Track job IDs and expire old jobs
- [ ] Implement reconnection logic
- [ ] Handle timestamp updates (if applicable)

### For Pool/Bridge Developers

- [ ] Implement TCP/SSL server listening
- [ ] Handle multiple concurrent miner connections
- [ ] Detect miner type from user agent (Standard vs BigJob format)
- [ ] Generate unique extranonce per connection
- [ ] Send `mining.set_difficulty` and `mining.set_extranonce` after subscription
- [ ] Validate `mining.authorize` credentials per policy
- [ ] Generate or fetch block templates
- [ ] Compute PrePowHash from block header
- [ ] Distribute jobs via `mining.notify` (appropriate format per client)
- [ ] Implement job storage (circular buffer, stale detection)
- [ ] Validate submitted shares (difficulty, format, duplicates, PoW)
- [ ] Implement job ID workaround for ASICs with wrong job IDs
- [ ] Track share statistics per miner
- [ ] For pools: Credit miner accounts for accepted shares
- [ ] For bridges: Submit valid blocks to blockchain node
- [ ] Handle connection drops and cleanup
- [ ] Implement error responses with appropriate codes
- [ ] Implement VarDiff (Variable Difficulty) if desired
- [ ] Handle timestamp validation

---

## References

- **JSON-RPC 2.0 Specification:** https://www.jsonrpc.org/specification
- **Stratum Protocol:** Based on EthereumStratum/1.0.0
- **kHeavyHash Algorithm:** See [kHeavyHash Technical Overview](kHeavyHash_Technical_Overview.md)
- **Kaspa Stratum Protocol Documentation:** https://manuals.plus/m/f20a9136b47412a5c58d207d7b7576e0cba08ff05cdbe492dce44e974c371d3e
- **IceRiver KS5L User Manual:** https://cdn.webshopapp.com/shops/353164/files/464331150/ks5l-operation-manual-en.pdf
- **NiceHash Kaspa Support:** https://www.nicehash.com/blog/post/kaspa-kheavyhash-is-now-supported-on-nicehash
- **kaspa-stratum-bridge (GitHub):** https://github.com/rdugan/kaspa-stratum-bridge
- **KStratum (GitHub):** https://github.com/KaffinPX/KStratum
- **Kaspa Wiki - Software Ecosystem:** https://wiki.kaspa.org/en/software-ecosystem
- **2Miners Stratum Protocols:** https://2miners.com/blog/nicehash-and-asic-compatible-stratum-protocols-on-2miners/
- **Bech32 Address Format:** BIP 173

---

**Document Version:** 2.0  
**Last Updated:** 2026-01-26  
**Maintained By:** Zorkcoin Documentation Project
