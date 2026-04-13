# KHeavyHash Stratum Protocol Specification

This document provides a complete specification of the Stratum mining protocol as implemented by the Zorkcoin Stratum Bridge. It is intended for developers implementing Stratum clients (miners) or Stratum servers (mining pools).

## Table of Contents

1. [Protocol Overview](#protocol-overview)
2. [Connection Establishment](#connection-establishment)
3. [Message Format](#message-format)
4. [Protocol Flow](#protocol-flow)
5. [Message Reference](#message-reference)
6. [Error Handling](#error-handling)
7. [Mining Algorithm](#mining-algorithm)
8. [Special Features](#special-features)
9. [Implementation Notes](#implementation-notes)

---

## Protocol Overview

The Stratum protocol is a JSON-RPC 2.0 based protocol used for communication between cryptocurrency miners and mining pools. This specification describes the KHeavyHash variant used for Zorkcoin mining.

### Key Characteristics

- **Transport**: TCP/IP (persistent connection)
- **Encoding**: JSON-RPC 2.0 (newline-delimited)
- **Message Format**: JSON objects, one per line
- **Connection**: Persistent until closed by either party
- **Encoding**: UTF-8

### Protocol Version

- **JSON-RPC Version**: 2.0
- **Stratum Version**: EthereumStratum/1.0.0 (standard format)
- **Extranonce Support**: Optional (for multi-client scenarios)

---

## Connection Establishment

### TCP Connection

1. Miner establishes TCP connection to server on configured port (default: 5555)
2. No initial handshake required - connection is immediately ready for JSON-RPC messages
3. Connection remains open until:
   - Miner disconnects
   - Server disconnects (error, timeout, or shutdown)
   - Network error occurs

### Connection Requirements

- **TCP_NODELAY**: Recommended for lower latency
- **Keep-Alive**: Optional but recommended
- **Timeout**: Server uses 5-second read deadlines
- **Buffer Size**: Messages typically < 1KB, but should support up to 32KB

---

## Message Format

### JSON-RPC 2.0 Structure

All messages follow JSON-RPC 2.0 format:

**Request (Miner → Server):**
```json
{
  "id": <number|string|null>,
  "jsonrpc": "2.0",
  "method": "<method_name>",
  "params": [<param1>, <param2>, ...]
}
```

**Response (Server → Miner):**
```json
{
  "id": <number|string|null>,
  "jsonrpc": "2.0",
  "result": <result_value>,
  "error": null
}
```

**Error Response (Server → Miner):**
```json
{
  "id": <number|string|null>,
  "jsonrpc": "2.0",
  "result": null,
  "error": [<error_code>, "<error_message>", <error_data>]
}
```

**Notification (Server → Miner, no ID):**
```json
{
  "jsonrpc": "2.0",
  "method": "<method_name>",
  "params": [<param1>, <param2>, ...]
}
```

### Message Delimiting

- Messages are newline-delimited (`\n`)
- Each message is a complete JSON object on a single line
- No multi-line JSON is supported
- Null bytes (`\x00`) in messages are stripped by server

### Message Ordering

- Requests and responses maintain request ID correlation
- Notifications can arrive at any time
- Server may send multiple notifications before a response
- Messages should be processed in order received

---

## Protocol Flow

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
  |   [wallet_address]            |
  |                               |
  |<-- Response ------------------|
  |   [true/false]                |
  |                               |
  |<-- mining.set_extranonce -----| (if extranonce enabled)
  |   [extranonce, size]          |
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
     │ 1. TCP Connect                              │
     │─────────────────────────────────────────────>│
     │                                              │
     │ 2. mining.subscribe                         │
     │─────────────────────────────────────────────>│
     │                                              │
     │ 3. Response: [true, "EthereumStratum/1.0.0"]│
     │<─────────────────────────────────────────────│
     │                                              │
     │ 4. mining.authorize                         │
     │─────────────────────────────────────────────>│
     │                                              │
     │ 5. Response: true                           │
     │<─────────────────────────────────────────────│
     │                                              │
     │ 6. mining.set_extranonce (if enabled)        │
     │<─────────────────────────────────────────────│
     │                                              │
     │ 7. mining.set_difficulty                    │
     │<─────────────────────────────────────────────│
     │                                              │
     │ 8. mining.notify                            │
     │<─────────────────────────────────────────────│
     │                                              │
     │ [Miner mines work...]                       │
     │                                              │
     │ 9. mining.submit                            │
     │─────────────────────────────────────────────>│
     │                                              │
     │ 10. Response: true                          │
     │<─────────────────────────────────────────────│
     │                                              │
     │ [Repeat steps 7-10 as new blocks arrive]    │
     │                                              │
```

---

## Message Reference

### mining.subscribe

**Direction**: Miner → Server  
**Type**: Request  
**Purpose**: Subscribe to mining notifications and identify miner software

**Request:**
```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "mining.subscribe",
  "params": ["<user_agent>"]
}
```

**Parameters:**
- `params[0]` (string, optional): User agent string identifying the miner software
  - Example: `"BzMiner/1.0.0"`, `"IceRiverMiner/2.0"`, `"VirtualMiner/1.0"`

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
  "result": [null, "<extranonce_hex>", <extranonce2_size>]
}
```

**Response Parameters:**
- Standard format:
  - `result[0]`: `true` (boolean)
  - `result[1]`: `"EthereumStratum/1.0.0"` (protocol version string)
- Bitmain format (when extranonce enabled):
  - `result[0]`: `null`
  - `result[1]`: Extranonce as hexadecimal string (e.g., `"00000001"`)
  - `result[2]`: Extranonce2 size in bytes (typically `8 - (extranonce_length / 2)`)

**Server Behavior:**
- Stores user agent in client context
- Detects Bitmain clients via regex: `.*(GodMiner).*`
- Assigns extranonce if extranonce is enabled (see [Extranonce](#extranonce-handling))
- Response format depends on client type and extranonce configuration

**Error Handling:**
- No specific errors for subscribe - always succeeds if connection is valid

---

### mining.authorize

**Direction**: Miner → Server  
**Type**: Request  
**Purpose**: Authenticate miner with wallet address

**Request:**
```json
{
  "id": 2,
  "jsonrpc": "2.0",
  "method": "mining.authorize",
  "params": ["<wallet_address>[.<worker_name>]", "<password>"]
}
```

**Parameters:**
- `params[0]` (string, required): Wallet address with optional worker name
  - Format: `"<address>"` or `"<address>.<worker>"`
  - Example: `"zorktest1abc123"` or `"zorktest1abc123.worker1"`
  - Address must be valid Bech32 format (zork1* or zorktest1*)
- `params[1]` (string, optional): Password (typically empty string `""`)

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
- Validates wallet address format (Bech32)
- Extracts worker name if present (format: `address.worker`)
- Caches address validation result
- Triggers work sending after successful authorization
- Sends extranonce notification if extranonce is enabled
- Disconnects client if address is invalid after 20 seconds

**Error Handling:**
- Invalid address format: Returns `false`, may disconnect after timeout
- Malformed request: Returns error response

**Post-Authorization:**
- Server immediately sends `mining.set_difficulty` (initial difficulty)
- Server sends `mining.notify` if cached block template exists
- Server sends extranonce notification if extranonce enabled

---

### mining.set_difficulty

**Direction**: Server → Miner  
**Type**: Notification (no ID)  
**Purpose**: Set or update mining difficulty

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
  - Example: `1.0`, `2.5`, `0.5`, `16.0`
  - Minimum: `0.0078125` (2^-7)
  - No maximum (but typically < 1000)

**Server Behavior:**
- Sent before first `mining.notify` (initial difficulty)
- Sent whenever difficulty changes (VarDiff)
- Difficulty applies to all subsequent shares until changed
- Miner must update difficulty immediately upon receipt

**Miner Behavior:**
- Must accept and apply difficulty immediately
- Difficulty applies to all shares submitted after this notification
- Should not submit shares below the new difficulty
- Old shares may still be submitted if they meet the new difficulty

**Timing:**
- Always sent before `mining.notify` for new jobs
- May be sent between jobs (VarDiff adjustment)
- Sent immediately after authorization (initial difficulty)

---

### mining.notify

**Direction**: Server → Miner  
**Type**: Notification (no ID)  
**Purpose**: Send new mining job to miner

**Standard Format (Most Miners):**
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

**BigJob Format (BzMiner/IceRiverMiner):**
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

**Standard Format Parameters:**
- `params[0]` (string): Job ID (unique identifier for this job)
  - Example: `"12345"`
  - Used in `mining.submit` to reference this job
- `params[1]` (array): PrePowHash as 4 uint64 values (little-endian)
  - Array of 4 numbers representing 32-byte PrePowHash
  - Example: `[0x0123456789abcdef, 0xfedcba9876543210, 0x1234567890abcdef, 0xfedcba0987654321]`
- `params[2]` (number): Unix timestamp (uint64)
  - Example: `1699123456`
  - Used in work header construction

**BigJob Format Parameters:**
- `params[0]` (string): Job ID (same as standard format)
- `params[1]` (string): Complete 80-byte work header as hexadecimal string
  - 160 hex characters (80 bytes × 2)
  - Format: `"0123456789abcdef..."` (160 chars)
  - Contains: PrePowHash (32 bytes) + Timestamp (8 bytes) + Padding (32 bytes) + Nonce (8 bytes, initially zero)

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

**Miner Behavior:**
- Must construct work header from job data
- Must increment nonce to find valid shares
- May update timestamp if needed (within valid range)
- Should submit shares using correct job ID

**Job Lifecycle:**
- Jobs are valid until:
  - New block is found (network advances)
  - Job becomes stale (> 8 blocks old)
  - Server explicitly invalidates job
- Miner should track multiple jobs and submit to correct one

---

### mining.submit

**Direction**: Miner → Server  
**Type**: Request  
**Purpose**: Submit a share (valid work result)

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
- `params[0]` (string): Worker name (optional, may be empty string `""`)
  - Typically matches worker name from `mining.authorize`
  - Used for statistics tracking
- `params[1]` (string): Job ID from `mining.notify`
  - Must match a valid, non-stale job
  - Example: `"12345"`
- `params[2]` (string): Nonce as hexadecimal string
  - 16 hex characters (8 bytes × 2)
  - Example: `"1234567890abcdef"`
  - May include `"0x"` prefix (automatically stripped by server)
  - If extranonce enabled, this is only the extranonce2 portion

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
2. Constructs full nonce (extranonce + extranonce2 if extranonce enabled)
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
- Server implements workaround: searches job history if share is invalid
- If correct job found, share is accepted with warning logged

---

### mining.extranonce.subscribe

**Direction**: Miner → Server  
**Type**: Request  
**Purpose**: Request extranonce notifications (optional)

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

**Direction**: Server → Miner  
**Type**: Notification (no ID)  
**Purpose**: Notify miner of extranonce (if extranonce enabled)

**Notification (Standard Format):**
```json
{
  "jsonrpc": "2.0",
  "method": "set_extranonce",
  "params": ["<extranonce_hex>", <extranonce2_size>]
}
```

**Notification (Bitmain Format):**
```json
{
  "jsonrpc": "2.0",
  "method": "mining.set_extranonce",
  "params": ["<extranonce_hex>", <extranonce2_size>]
}
```

**Parameters:**
- `params[0]` (string): Extranonce as hexadecimal string
  - Example: `"00000001"`
  - Length depends on extranonce size configuration
- `params[1]` (number): Extranonce2 size in bytes
  - Typically: `8 - (extranonce_length / 2)`
  - Example: If extranonce is 2 bytes (4 hex chars), extranonce2_size = 6

**Server Behavior:**
- Sent after successful `mining.authorize` if extranonce is enabled
- Method name depends on client type (Bitmain vs standard)
- Extranonce is unique per client connection
- Full nonce = extranonce + extranonce2 (miner only sends extranonce2)

**Miner Behavior:**
- Must prepend extranonce to all nonces before hashing
- Only sends extranonce2 portion in `mining.submit`
- Server automatically prepends extranonce during validation

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
- Error 23 indicates invalid difficulty
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

---

## Mining Algorithm

### kHeavyHash Overview

KHeavyHash is the proof-of-work algorithm used by Zorkcoin. It operates on an 80-byte work header.

### Work Header Construction

**From Standard Format:**
1. Extract PrePowHash from params[1] (4 uint64 values → 32 bytes)
2. Extract timestamp from params[2] (uint64)
3. Construct 80-byte header:
   - Bytes 0-31: PrePowHash
   - Bytes 32-39: Timestamp (little-endian uint64)
   - Bytes 40-71: Padding (32 zero bytes)
   - Bytes 72-79: Nonce (little-endian uint64, initially zero)

**From BigJob Format:**
1. Decode hex string directly (160 hex chars = 80 bytes)
2. Header is already complete

### PrePowHash Calculation

PrePowHash is calculated from the block header with timestamp=0 and nonce=0:

```
Header (88 bytes):
- Version: 4 bytes (int32, little-endian)
- PrevBlock: 32 bytes (Hash)
- MerkleRoot: 32 bytes (Hash)
- Timestamp: 8 bytes (uint64, set to 0)
- Bits: 4 bytes (uint32, little-endian)
- Nonce: 8 bytes (uint64, set to 0)

PrePowHash = SHA256(SHA256(Header))
```

### kHeavyHash Algorithm

1. **Input**: 80-byte work header
2. **Process**:
   - Split into 8×10 byte matrix
   - Apply AES-128 operations to each row
   - Mix results using matrix operations
   - Final SHA-256 hash
3. **Output**: 32-byte hash

### Difficulty Checking

1. **Reverse hash bytes** (Bitcoin-style, big-endian comparison):
   ```python
   reversed_hash = reverse_bytes(hash)
   ```

2. **Compare against target**:
   ```python
   target = max_target / difficulty
   pow_value = int(reversed_hash, 16)
   if pow_value <= target:
       # Valid share!
   ```

3. **Difficulty levels**:
   - **Stratum difficulty**: Minimum difficulty for share acceptance
   - **Network difficulty**: Difficulty required for block submission
   - Share must meet stratum difficulty
   - Block must meet network difficulty

### Nonce Incrementing

- Nonce is 8 bytes (64 bits)
- Miner increments nonce from 0 to 2^64 - 1
- If extranonce enabled:
  - Full nonce = extranonce (server-assigned) + extranonce2 (miner-assigned)
  - Miner only increments extranonce2 portion
  - Server prepends extranonce during validation

### Timestamp Updates

- Timestamp can be updated by miner (within valid range)
- Server validates timestamp is reasonable
- Timestamp affects hash result
- Miner may update timestamp to find valid shares

---

## Special Features

### Extranonce Handling

**Purpose**: Allow multiple clients to mine without nonce collisions

**How It Works:**
1. Server assigns unique extranonce to each client
2. Full nonce = extranonce + extranonce2
3. Miner only sends extranonce2 in `mining.submit`
4. Server prepends extranonce before validation

**Configuration:**
- Extranonce size: Configurable (typically 0-3 bytes)
- Max clients: 2^(8*extranonce_size) - 1
- Extranonce2 size: 8 - extranonce_size bytes

**Example:**
```
Extranonce size: 1 byte (2 hex chars)
Extranonce: "01"
Extranonce2 size: 7 bytes (14 hex chars)

Miner finds nonce: "1234567890abcdef"
Full nonce: "01" + "1234567890abcdef" = "011234567890abcdef"
Miner submits: "1234567890abcdef" (only extranonce2)
```

### Variable Difficulty (VarDiff)

**Purpose**: Automatically adjust difficulty to target share submission rate

**How It Works:**
1. Server monitors share submission rate per client
2. Compares against target rate (e.g., 20 shares/minute)
3. Adjusts difficulty up or down based on actual rate
4. Sends `mining.set_difficulty` notification when adjustment needed

**Windows:**
- 1 minute: Initial check
- 3 minutes: Second stage
- 10 minutes: Third stage
- 30 minutes: Fourth stage
- 60 minutes: Fifth stage
- 240 minutes: Sixth stage
- Continuous: Final stage

**Tolerances:**
- Each window has tolerance (e.g., ±50% for 3-minute window)
- If rate exceeds tolerance, difficulty is adjusted
- Adjustment formula: `new_diff = old_diff * (actual_rate / target_rate)`

**Clamping:**
- Difficulty can be clamped to powers of 2
- Minimum difficulty: 0.0078125 (2^-7)
- Maximum difficulty: No hard limit (typically < 1000)

### BigJob Format

**Purpose**: Support ASICs that require complete work header as hex string

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

**Problem**: Some ASICs (IceRiver, Bitmain) submit shares with wrong job IDs

**Solution**: Server searches job history if share is invalid
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

**Line Delimiting:**
- Read until newline (`\n`)
- Handle partial messages (buffer until complete)
- Maximum message size: 32KB (safety limit)

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

**Client State:**
- Current job ID
- Current difficulty
- Extranonce (if enabled)
- Connection status

### Job Management

**Job Storage:**
- Circular buffer (32 jobs)
- Job ID modulo 32 for indexing
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
3. Construct full nonce (extranonce + extranonce2)
4. Calculate block hash using kHeavyHash
5. Check share meets stratum difficulty
6. Check for duplicates
7. If meets network difficulty, submit block
8. Record statistics

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
if int(reversed_hash, 16) <= target:
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

**Address Validation:**
- Validate Bech32 format
- Check network prefix (zork1* or zorktest1*)
- Cache validation results
- Disconnect invalid addresses

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

## Appendix

### Example Message Exchange

**Complete Connection Flow:**

```
1. TCP Connect
   Miner → Server: [TCP connection established]

2. Subscribe
   Miner → Server:
   {
     "id": 1,
     "jsonrpc": "2.0",
     "method": "mining.subscribe",
     "params": ["VirtualMiner/1.0"]
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
     "params": ["zorktest1abc123.worker1", ""]
   }
   
   Server → Miner:
   {
     "id": 2,
     "jsonrpc": "2.0",
     "result": true
   }

4. Set Difficulty
   Server → Miner:
   {
     "jsonrpc": "2.0",
     "method": "mining.set_difficulty",
     "params": [1.0]
   }

5. Notify (Standard Format)
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

6. Submit
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

## References

- **JSON-RPC 2.0 Specification**: https://www.jsonrpc.org/specification
- **Stratum Protocol**: Based on EthereumStratum/1.0.0
- **kHeavyHash Algorithm**: Zorkcoin-specific PoW algorithm
- **Bech32 Address Format**: BIP 173

---

**Document Version**: 1.0  
**Last Updated**: 2026-01-25  
**Maintained By**: Zorkcoin Stratum Bridge Project
