# Data Plane: Workers + Queues

The data plane handles all ciphertext transport with a focus on low-latency delivery.

---

## Application Messages

Message flow prioritizes delivery speed:

1. Client encrypts message using MLS application keys
2. Worker receives ciphertext over WebSocket or HTTP
3. Worker enqueues to `MLS_QUEUE` for ordered delivery
4. Queue consumer delivers to Room DO
5. Room DO broadcasts to WebSocket clients (synchronous, non-blocking)
6. Room DO buffers message in rolling history buffer
7. Room DO enqueues to `HISTORY_QUEUE` (fire-and-forget)
8. History queue consumer writes to R2 in batches

Properties:

- **Delivery-first architecture** — clients receive messages before R2 persistence
- **No plaintext** — server sees only ciphertext
- **No handle or identity leakage** — only pseudonym IDs in routing
- **High-throughput, low-latency global delivery**

---

## Message Delivery Flow

```mermaid
sequenceDiagram
    autonumber
    participant C as Client (Sender)
    participant W as Worker (Ingress)
    participant MQ as MLS_QUEUE
    participant DO as Room DO
    participant HQ as HISTORY_QUEUE
    participant R2 as R2 Bucket
    participant M as Member Clients

    Note over C: MLS encrypts message<br/>→ ciphertext only

    C->>W: WebSocket send(ciphertext)
    W->>W: Validate envelope structure
    W->>MQ: Enqueue message

    MQ->>DO: Deliver in order

    DO->>M: WebSocket broadcast (sync)
    DO->>DO: Buffer in rolling history
    DO-->>HQ: Enqueue for R2 (fire-and-forget)

    Note over M: Clients receive message<br/>immediately

    HQ->>R2: Batched write
    
    Note over R2: Message persisted<br/>for reconnection
```

The critical insight is that **R2 writes happen asynchronously** after message delivery. This ensures:

- Clients receive messages with minimal latency
- R2 write failures don't impact delivery
- Batched writes reduce R2 API costs

---

## Queue Architecture

tunnl3d uses three Cloudflare Queues to separate concerns and optimize for latency:

| Queue | Purpose | Batch Size | Timeout | Retries |
|-------|---------|------------|---------|---------|
| `MLS_QUEUE` | Real-time message delivery to Room DOs | 100 | 1s | 3 |
| `HISTORY_QUEUE` | Async R2 persistence (off hot path) | 100 | 5s | 5 |
| `MLS_DLQ` | Dead letter queue for failed messages | — | — | — |

This separation ensures that R2 write latency never impacts message delivery latency.

---

## Message History & Client Reconnection

tunnl3d provides durable message history for client reconnection with a two-tier architecture:

1. **DO Buffer** — Rolling buffer of last 50 messages for instant access
2. **R2 Storage** — Durable persistence for full history replay

This design ensures clients can immediately access recent messages even before R2 persistence completes.

### History Architecture

```mermaid
flowchart TD
    subgraph Request["History Request"]
        C[Client]
    end
    
    subgraph HW["History Worker"]
        Q1[Query R2 for<br/>persisted msgs]
        Q2[Query DO buffer<br/>for recent msgs]
        M[Merge & Deduplicate]
    end
    
    subgraph Storage["Storage Layer"]
        R2[(R2 Bucket)]
        DO[(Room DO Buffer)]
    end
    
    C --> HW
    Q1 --> R2
    Q2 --> DO
    Q1 --> M
    Q2 --> M
    M --> C
```

The History Worker intelligently combines data from both sources:

- Always queries R2 for persisted messages
- Queries DO buffer when the time range is recent (within 30 seconds)
- Deduplicates by message ID (R2 messages are authoritative)
- Returns sorted, unified results

### DO Rolling Buffer

The Room DO maintains a rolling buffer of the 50 most recent messages:

```typescript
interface BufferedHistoryEntry {
  id: string;           // Message ID
  sequence: number;     // Client-assigned sequence
  timestamp: string;    // ISO 8601 timestamp
  envelope: AnyEnvelope;
  bufferedAt: string;   // When buffered in DO
}
```

Buffer properties:

| Property | Value |
|----------|-------|
| Size | 50 messages |
| Eviction | FIFO (oldest removed when full) |
| Latency | Immediate (DO storage) |
| Durability | Transient (for recent messages only) |

The buffer is populated synchronously during message delivery, ensuring messages are available for history queries immediately.

### R2 Persistent Storage

Messages are durably persisted to R2 via the `HISTORY_QUEUE`:

#### R2 Key Structure

```
messages/{tenant_id}/{room_id}/{YYYY-MM-DD}/{timestamp}-{message_id}.json
```

Example:
```
messages/tenant-123/room-456/2024-12-23/1703337600000-msg-abc123.json
```

#### Message Object Format

```json
{
  "id": "msg-abc123",
  "sequence": 42,
  "timestamp": "2024-12-23T12:00:00.000Z",
  "envelope": {
    "type": "mls_app",
    "tenantId": "tenant-123",
    "roomId": "room-456",
    "pseudonymId": "user-789",
    "sequence": 42,
    "timestamp": "2024-12-23T12:00:00.000Z",
    "ciphertext": "base64-encoded-encrypted-content"
  },
  "storedAt": "2024-12-23T12:00:01.000Z"
}
```

#### Storage Properties

| Property | Value |
|----------|-------|
| Cost | $0.015/GB/month |
| Max object size | 5 TB |
| Default retention | 7 days |
| Auto-expiration | Lifecycle policy |
| Write pattern | Batched (via HISTORY_QUEUE) |

#### Batched Write Flow

```mermaid
sequenceDiagram
    participant DO as Room DO
    participant HQ as HISTORY_QUEUE
    participant HC as History Consumer
    participant R2 as R2 Bucket

    DO-->>HQ: Enqueue message (fire-and-forget)
    
    Note over HQ: Batches accumulate<br/>(up to 100 msgs, 5s timeout)
    
    HQ->>HC: Deliver batch
    
    par Parallel writes
        HC->>R2: PUT message 1
        HC->>R2: PUT message 2
        HC->>R2: PUT message N
    end
    
    HC->>HQ: ACK batch
```

Benefits of batched writes:

- Reduced R2 API calls
- Lower cost
- Better throughput
- Failures don't block delivery

---

## History API

The History Worker serves the `/api/v1/history` endpoint:

**Endpoint:** `GET /api/v1/history`

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenant` | string | Yes | Tenant ID |
| `room` | string | Yes | Room ID |
| `since` | ISO 8601 | Yes | Start timestamp (inclusive) |
| `sequence` | number | No | Last known sequence (filters results) |
| `limit` | number | No | Max messages (default: 100, max: 1000) |
| `cursor` | string | No | Pagination cursor for next page |

**Response:**

```json
{
  "messages": [
    {
      "id": "msg-abc123",
      "sequence": 43,
      "timestamp": "2024-12-23T10:05:00.000Z",
      "envelope": { ... },
      "storedAt": "2024-12-23T10:05:01.000Z"
    }
  ],
  "hasMore": true,
  "nextCursor": "2024-12-23T10:06:00.000Z:msg-def456"
}
```

**Error Responses:**

| Status | Description |
|--------|-------------|
| 400 | Missing required parameters |
| 401 | Missing or expired access token |
| 403 | Invalid access token |
| 429 | Rate limited |
| 500 | Internal error |

### Query Logic

```typescript
// Pseudocode for history query
async function queryHistory(tenant, room, since, sequence, limit) {
  // 1. Always query R2
  const r2Messages = await readFromR2(tenant, room, since, sequence, limit);
  
  // 2. Check if we need DO buffer
  const now = Date.now();
  const sinceTimestamp = new Date(since).getTime();
  const recentThreshold = now - 30_000; // 30 seconds
  
  if (sinceTimestamp >= recentThreshold || r2Messages.length === 0) {
    // Query DO buffer for recent messages
    const bufferMessages = await queryDoBuffer(tenant, room, since, sequence);
    
    // Merge and deduplicate
    return mergeAndDeduplicate(r2Messages, bufferMessages);
  }
  
  return r2Messages;
}
```

---

## Client Reconnection Flow

When a client reconnects after being offline:

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant HW as History Worker
    participant DO as Room DO
    participant R2 as R2 Bucket

    Note over C: Client comes online<br/>after being offline

    C->>HW: GET /api/v1/history
    
    par Query both sources
        HW->>R2: Query persisted messages
        R2-->>HW: Return matching messages
        HW->>DO: Query recent buffer<br/>(if recent time range)
        DO-->>HW: Return buffered messages
    end
    
    HW->>HW: Merge & deduplicate
    HW->>C: Return unified history

    Note over C: Process messages<br/>(MLS decrypt)

    C->>DO: GET /ws/:tenant/:room
    DO->>C: 101 Switching Protocols

    Note over C,DO: Resume real-time<br/>WebSocket messages
```

### Client State Tracking

Clients persist locally:

```typescript
interface ClientRoomState {
  tenantId: string;
  roomId: string;
  lastSequence: number;        // Last received sequence number
  lastTimestamp: string;       // ISO 8601 timestamp of last message
  lastMessageId: string;       // ID of last received message
}
```

### Edge Cases

| Scenario | Handling |
|----------|----------|
| **Duplicate messages** | Client filters by sequence number; server deduplicates by ID |
| **Out-of-order messages** | Server sorts by sequence before returning |
| **Large gap** | Paginate with cursor, process in batches |
| **MLS epoch change** | Client may need to re-fetch group state |
| **Network failure during replay** | Retry from last processed sequence |
| **R2 unavailable** | DO buffer provides recent messages; return 503 for older |
| **Message in buffer but not R2** | Merge logic includes buffer messages not yet in R2 |

---

## History API Security

### Access Control

The history API requires HMAC-signed access tokens to prevent unauthorized access.

**Protections:**

- **Tenant/room enumeration** — Attackers cannot probe for valid IDs
- **Metadata leakage** — Only authenticated room members can see message metadata
- **Traffic analysis** — Timing and frequency patterns are protected

### Token Format

```
token = base64(HMAC-SHA256(tenant:room:pseudonymId:expiry, access_key))
```

### Client-Side Token Generation

Clients derive access tokens using MLS exporter (zero-trust, no server secrets):

```typescript
// Derive access key from MLS group state
const accessKey = await mlsGroup.export("tunnl3d-history-access", roomId, 32);

// Build message to sign
const expiry = Date.now() + 3600000; // 1 hour
const message = `${tenantId}:${roomId}:${pseudonymId}:${expiry}`;

// Compute HMAC and encode
const token = base64(hmacSha256(accessKey, message));
```

### Request Format

```
GET /api/v1/history
  ?tenant=xxx
  &room=yyy
  &since=2024-12-23T00:00:00.000Z
  &pseudonym=user-789
  &expiry=1735000000000
  &token=base64-hmac-signature

# Or via Authorization header:
Authorization: Bearer base64-hmac-signature
```

### Security Properties

| Property | Guarantee |
|----------|-----------|
| **Authentication** | Only room members with MLS state can generate valid tokens |
| **Authorization** | Token is scoped to specific tenant/room |
| **Expiry** | Tokens are time-limited (recommended: 1 hour) |
| **Non-replayable** | Expired tokens rejected |
| **Zero-trust** | Server validates, but client holds key derivation secrets |
| **Timing-safe** | Constant-time comparison prevents timing attacks |

---

## Message Retention & Lifecycle

### Default Retention Policy

- Messages older than 7 days are automatically deleted
- Configurable per tenant via lifecycle rules

### R2 Lifecycle Configuration

Lifecycle rules must be configured via Cloudflare Dashboard or API (not supported in wrangler.toml):

```json
{
  "rules": [
    {
      "id": "expire-old-messages",
      "enabled": true,
      "conditions": {
        "prefix": "messages/"
      },
      "actions": {
        "expiration": {
          "days": 7
        }
      }
    }
  ]
}
```

To apply via API:

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/{account_id}/r2/buckets/{bucket_name}/lifecycle" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  -d '{"rules":[{"id":"expire-old-messages","enabled":true,"conditions":{"prefix":"messages/"},"actions":{"expiration":{"days":7}}}]}'
```

### Rate Limiting

| Limit | Value |
|-------|-------|
| Requests per minute per tenant | 60 |
| Max messages per request | 1000 |
| Max time range | 7 days |
