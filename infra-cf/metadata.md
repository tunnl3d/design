# Metadata Plane: Encrypted Durable Object Storage

Durable Objects store all room state, but sensitive information is always encrypted client-side before storage. Cloudflare acts as a blind storage substrate.

---

## Stored Data Categories

Durable Objects store:

- Encrypted room metadata
- Encrypted quorum state
- Encrypted admin payloads
- Recent message buffer
- Routing metadata (plaintext)

---

## Storage Keys

```typescript
const STORAGE_KEYS = {
  ROOM_STATE: "room:state",
  PROPOSAL_PREFIX: "proposal:",
  ONBOARDING_PREFIX: "onboarding:",
  HISTORY_BUFFER: "history:buffer",
};
```

---

## MLS‑Derived Storage Keys

Clients derive storage keys using MLS exporter:

### Stable Key (Default)

```
K_storage = MLS-Exporter("tunnl3d-do-metadata", "group-creation", 32)
```

- Derived once at group creation
- Shared by all group members
- Used for metadata that should survive epoch changes

### Per‑Epoch Key (High‑Security)

```
K_storage(E) = MLS-Exporter("tunnl3d-do-metadata", "epoch-" || E, 32)
```

- Derived per MLS epoch
- Provides forward secrecy for metadata
- Old metadata becomes unreadable after epoch rotation

---

## Encrypted Metadata

### What is Encrypted

- Room name
- Description
- Roles
- Admin notes
- Display attributes
- Any sensitive configuration

### What Remains Plaintext (Required for Routing)

- `tenant_id`
- `room_id`
- `pseudonym_id`
- `proposal_id`
- Queue pointers

**Cloudflare cannot decrypt any metadata.**

---

## Room State Structure

```typescript
interface RoomState {
  /** Tenant ID (plaintext, for routing) */
  tenantId: string;
  /** Room ID (plaintext, for routing) */
  roomId: string;
  /** Encrypted room metadata (name, description, roles, etc.) */
  encryptedMetadata?: string;
  /** Encrypted quorum state */
  encryptedQuorumState?: string;
  /** Active proposal IDs (plaintext, for routing) */
  activeProposals: string[];
  /** Created timestamp */
  createdAt: string;
  /** Last modified timestamp */
  modifiedAt: string;
}
```

---

## Proposal State Structure

```typescript
interface ProposalState {
  /** Proposal ID (plaintext, for routing) */
  proposalId: string;
  /** Proposer pseudonym ID */
  proposerPseudonymId: string;
  /** Encrypted proposal content */
  encryptedContent: string;
  /** Encrypted approvals (as array of ciphertexts) */
  encryptedApprovals: string[];
  /** Status (server only knows if committed, not quorum details) */
  status: "pending" | "committed" | "rejected" | "expired";
  /** Created timestamp */
  createdAt: string;
  /** Expires timestamp */
  expiresAt: string;
}
```

---

## Onboarding Slot Structure

```typescript
interface OnboardingSlot {
  /** Slot ID */
  slotId: string;
  /** Onboarding public key (Phase 1) */
  onboardingPublicKey?: string;
  /** Encrypted admin payload (Phase 2) */
  encryptedAdminPayload?: string;
  /** Phase status */
  phase: "client_acceptance" | "admin_completion" | "client_finalization";
  /** Created timestamp */
  createdAt: string;
  /** Expires timestamp */
  expiresAt: string;
}
```

---

## Security Properties

### What Cloudflare Can See

- Storage key names (e.g., `room:state`, `proposal:abc123`)
- Ciphertext blobs (opaque, no structure visible)
- Timestamps
- Routing identifiers

### What Cloudflare Cannot See

- Room names or descriptions
- Member handles or identities
- Quorum configuration
- Who approved what
- Proposal contents
- Any semantic metadata

---

## Storage Patterns

### Read Pattern

1. Client requests encrypted blob via DO HTTP endpoint
2. DO retrieves from storage, returns ciphertext
3. Client decrypts locally with MLS-derived key

### Write Pattern

1. Client encrypts data with MLS-derived key
2. Client sends ciphertext to DO via HTTP
3. DO stores opaque blob
4. DO never attempts decryption

### Consistency

- DO storage is strongly consistent within a single DO
- All operations on a room go through the same DO instance
- No cross-room transactions needed
