# tunnl3d Architecture Specification  
Version: 2.0 (Cloudflare‑Native, Client‑Governed MLS)  
Revised: Client‑Side Quorum, Encrypted Metadata, Zero‑Trust Infrastructure

This document defines the tunnl3d architecture as a **zero‑API, MLS‑secure, server‑blind messaging platform** built on Cloudflare’s global edge network.  
All cryptographic authority resides in **clients**, not servers.  
Cloudflare provides **transport, routing, ordering, and persistence**, but never sees plaintext or group semantics.

---

# 1. High‑Level Overview

tunnl3d is composed of:

- **Cloudflare Workers** — ingress/egress, routing, validation of message envelopes  
- **Cloudflare Queues** — ordered, durable transport for MLS control and quorum updates  
- **Cloudflare Durable Objects (DOs)** — blind coordination substrate for:
  - encrypted quorum state  
  - encrypted room metadata  
  - proposal lifecycle  
  - routing and ordering  
- **Cloudflare KV / R2** — optional encrypted blob storage  
- **Stripe** — billing + tenant identity oracle  
- **Clients (WASM + MAUI)** — MLS core, encryption, quorum logic, UI, onboarding, recovery

Key properties:

- **No server‑side MLS state**  
- **No server‑side plaintext**  
- **No server‑side authority**  
- **Client‑side quorum governs membership**  
- **All sensitive metadata encrypted with MLS‑derived keys**  
- **Cloudflare is fully untrusted and replaceable**

---

# 2. Data Plane: Workers + Queues

The data plane handles all ciphertext transport.

## 2.1 Application Messages

- Clients encrypt messages using MLS application keys.  
- Workers receive ciphertext over WebSockets/HTTP.  
- Workers enqueue messages into Queues for fan‑out.  
- Other Workers deliver ciphertext to subscribed clients.  
- No plaintext is visible to Cloudflare.

Topics:

    tenant/{tenant_id}/room/{room_id}/mls/app

Properties:

- No message history stored server‑side  
- No plaintext  
- No handle or identity leakage  
- High‑throughput, low‑latency global delivery

---

# 3. Control Plane: Workers + Queues + Durable Objects

The control plane handles:

- MLS control messages  
- Quorum‑based membership changes  
- Encrypted room metadata  
- Proposal lifecycle  

## 3.1 MLS Control Messages

Flow:

1. Client publishes MLS control envelope (proposal, commit, GroupInfo).  
2. Worker validates envelope structure (not cryptography).  
3. Worker enqueues into Queue.  
4. DO processes ordered control messages:
   - updates encrypted metadata  
   - stores encrypted quorum state  
   - routes commit notifications  

DO never sees plaintext or MLS secrets.

Topics:

    tenant/{tenant_id}/room/{room_id}/mls/control

---

# 4. Governance Plane: Client‑Side Quorum

tunnl3d uses **client‑side quorum** for all structural changes:

- add member  
- remove member  
- change room configuration  
- change approver set  
- change quorum threshold  

## 4.1 Approver Set (A)

- Defined per room  
- Typically 3–7 members (normal)  
- 5–9 members (high‑security)  

## 4.2 Quorum Threshold (k)

Default:

    If |A| ≤ 3         → k = 2  
    If |A| ∈ [4,5]     → k = 3  
    If |A| ≥ 6         → k = ceil(2/3 * |A|)

Whole‑group mode (optional):

    k = max(3, ceil(0.2 * n))

## 4.3 Quorum State

All quorum state is:

- constructed client‑side  
- encrypted with MLS‑derived storage keys  
- stored as opaque blobs in DOs  
- never visible to Cloudflare  

Clients:

- fetch encrypted quorum state  
- decrypt  
- verify approvals  
- detect quorum  
- publish MLS commits  

Cloudflare never knows:

- who approved  
- how many approved  
- when quorum is reached  

---

# 5. Metadata Plane: Encrypted Durable Object Storage

Durable Objects store:

- encrypted room metadata  
- encrypted quorum state  
- encrypted admin payloads  
- routing metadata (plaintext)  

## 5.1 MLS‑Derived Storage Keys

Clients derive storage keys using MLS exporter:

Stable key (default):

    K_storage = MLS-Exporter("tunnl3d-do-metadata", "group-creation", 32)

Per‑epoch key (high‑security):

    K_storage(E) = MLS-Exporter("tunnl3d-do-metadata", "epoch-" || E, 32)

## 5.2 Encrypted Metadata

Encrypted:

- room name  
- description  
- roles  
- admin notes  
- display attributes  
- any sensitive configuration  

Plaintext (required for routing):

- tenant_id  
- room_id  
- pseudonym_id  
- proposal_id  
- Queue pointers  

Cloudflare cannot decrypt any metadata.

---

# 6. Onboarding: Two‑Phase, Client‑Side Handles, Encrypted Payloads

Onboarding remains:

- two‑phase  
- client‑encrypted  
- server‑blind  

## 6.1 Phase 1 — Client Acceptance

Client sends:

- slot_id  
- onboarding_public_key  
- optional invite challenge  

Workers/DOs:

- validate slot_id  
- store onboarding_public_key  
- never see handles or PII  

## 6.2 Phase 2 — Admin Completion

Admin encrypts:

- handle  
- pseudonym_id  
- display attributes  

…to the onboarding_public_key.

DO stores ciphertext only.

## 6.3 Phase 3 — Client Finalisation

Client decrypts admin payload and joins MLS group.

Server never sees:

- handle  
- mapping  
- onboarding secrets  

---

# 7. Tenant Identity: Stripe as Oracle

Stripe stores:

- tenant_id  
- tenant_hmac  
- tenant_recovery_id  

Workers validate:

1. Stripe webhook signature  
2. tenant_hmac = HMAC(secret, tenant_id)  

No PII stored in tunnl3d.

---

# 8. Recovery: Encrypted Blobs

Recovery blobs contain:

- admin keys  
- pseudonym mappings  
- bootstrap metadata  

Encrypted client‑side with recovery key.

Stored in DO or R2 as ciphertext.

Server never sees decrypted recovery data.

---

# 9. Security Properties

## 9.1 End‑to‑End Encryption

- MLS encrypts all application messages  
- Only clients hold keys  
- Cloudflare sees ciphertext only  

## 9.2 Zero‑Trust Infrastructure

Cloudflare cannot:

- decrypt messages  
- decrypt metadata  
- forge approvals  
- detect quorum  
- add ghost members  
- remove members  
- rewrite history  

## 9.3 Client‑Side Governance

- Quorum enforced cryptographically  
- DO is blind  
- State actor must compromise k clients  

## 9.4 Metadata Confidentiality

- All semantic metadata encrypted  
- Only routing identifiers plaintext  

## 9.5 Forward Secrecy & PCS

MLS provides:

- forward secrecy  
- post‑compromise security  
- epoch‑based key rotation  

---

# 10. Operational Model

## 10.1 Cloudflare Responsibilities

- Transport  
- Routing  
- Ordering  
- Persistence  
- Stripe webhook ingress  
- Zero plaintext handling  

## 10.2 Client Responsibilities

- MLS core  
- Encryption/decryption  
- Quorum logic  
- Metadata encryption  
- Onboarding crypto  
- Recovery crypto  
- UI  

## 10.3 No Server‑Side AS

All authority resides in clients.

Cloudflare is a **stateless, untrusted substrate**.

---

# 11. Data Flow

```mermaid
sequenceDiagram
    autonumber

    participant C as Client (Sender)
    participant W_in as Worker (Ingress)
    participant Q as Cloudflare Queue
    participant DO as Durable Object (Room Router)
    participant W_out as Worker (Egress)
    participant M as Member Clients (Receivers)

    Note over C: MLS encrypts message<br/>→ ciphertext only

    C->>W_in: WebSocket send(ciphertext)
    W_in->>W_in: Validate envelope (no plaintext)
    W_in->>Q: Enqueue ciphertext<br/>tenant/{tenant}/room/{room}/mls/app

    Note over Q: Durable, ordered, global transport

    Q->>DO: Deliver ciphertext in order

    DO->>DO: Lookup connected WebSocket sessions<br/>(subscription registry)
    DO->>W_out: Forward ciphertext to egress worker

    Note over W_out: No decryption, no MLS state

    W_out->>M: WebSocket push(ciphertext)

    Note over M: MLS decrypts<br/>→ plaintext rendered locally
```

---

# 12. Summary

tunnl3d on Cloudflare is:

- **Zero‑trust**  
- **Zero‑plaintext**  
- **Zero‑API**  
- **Client‑governed**  
- **Cryptographically self‑authenticating**  
- **Economically minimal**  
- **Resilient to state‑actor coercion**  

Cloudflare provides global transport and coordination.  
Clients provide all cryptographic correctness and governance.

This architecture achieves maximal privacy, maximal resilience, and minimal operational cost.