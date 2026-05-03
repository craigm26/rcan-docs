# §17 Distributed Registry Node Protocol

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** This section defines the Distributed Registry Node Protocol — the layer that enables multiple independent RCAN registry nodes to interoperate as a coherent global namespace. It specifies how nodes identify themselves, partition the RRN namespace, synchronise records, and establish a verifiable trust chain rooted at `rcan.dev`.

---

## 17.1 Node Types

The RCAN distributed registry defines four node types, each with distinct authority level, namespace scope, and operational responsibilities.

| Node Type | Namespace Authority | Signs Records | Delegation Cert | Syncs To |
|-----------|--------------------|--------------|--------------------|---------|
| **Root** | Global (all RRNs) | Yes | Self (trust anchor) | — |
| **Authoritative** | Delegated prefix (e.g. `BD`) | Yes | Required (from root) | Root |
| **Resolver** | None | No | Not required | Authoritative / Root |
| **Cache** | None | No | Not required | Authoritative (TTL-bound) |

**Root** (`rcan.dev`): Operated by the RCAN Working Group. Authoritative for the global namespace, delegation authority, and the root of the trust chain. Maintains `namespace_delegations` table; issues delegation certificates to Authoritative nodes; resolves legacy `RRN-XXXXXXXX` RRNs directly.

**Authoritative** (manufacturer-run): Operated by a robot manufacturer or organisation. Authoritative for a delegated RRN prefix namespace (e.g. `BD`, `UR`). Holds a delegation cert signed by root; signs all records it registers; syncs records upstream to root on schedule. MUST expose `/.well-known/rcan-node.json`.

**Resolver** (fleet operator): No namespace authority — caches and proxies records from Authoritative nodes and root. No delegation cert required. Caches records with TTL enforcement; serves local fleet without network dependency. MUST NOT register new RRNs in authoritative namespaces.

**Cache** (edge/CDN): Stores only TTL-bound records. Cannot originate records. Serves as a performance layer in front of Authoritative nodes. Transparent on cache miss (proxies upstream). MUST include `X-RCAN-Cache: HIT|MISS` header. MUST NOT serve stale records beyond 2× TTL.

---

## 17.2 RRN Namespace Partitioning

The Robot Registration Number (RRN) namespace is partitioned to support delegation to Authoritative nodes while preserving backwards compatibility with legacy RRNs.

### Legacy RRNs

Legacy RRNs use the format `RRN-XXXXXXXX` (8 uppercase hex characters). These RRNs were assigned before namespace delegation was introduced and remain registered at root. They continue to resolve unchanged.

```
# Legacy format (root-only)
RRN-000000000042  → resolves at rcan.dev
RRN-DEADBEEF      → resolves at rcan.dev
```

### Delegated RRNs

Delegated RRNs use the format `RRN-{PREFIX}-{SEQUENCE}`, where:

- `PREFIX` — 2 to 6 uppercase ASCII letters (A–Z), uniquely assigned by root to an Authoritative node.
- `SEQUENCE` — Zero-padded decimal sequence number (minimum 8 digits), assigned by the Authoritative node.

```
# Delegated format (authoritative node)
RRN-BD-00000001    → Boston Dynamics, resolves at registry.boston-dynamics.com
RRN-UR-00000042    → Universal Robots, resolves at registry.universal-robots.com
RRN-SPOT-00000001  → prefix "SPOT", resolves at delegated node
```

### Namespace Delegations Table

Root maintains a `namespace_delegations` table mapping each PREFIX to its Authoritative node URL and delegation certificate. This table is publicly queryable:

```
GET https://rcan.dev/api/rcan/v1/delegations
GET https://rcan.dev/api/rcan/v1/delegations/{prefix}
```

| Field | Type | Description |
|-------|------|-------------|
| `prefix` | string (2–6 chars) | Uppercase ASCII prefix (e.g. `BD`) |
| `node_url` | string (URL) | Authoritative node base URL |
| `operator` | string | Organisation holding this delegation |
| `delegated_at` | ISO 8601 | When the delegation was granted |
| `cert_fingerprint` | string (`sha256:...`) | Fingerprint of the delegation cert |

---

## 17.3 Node Identity

Every RCAN registry node (Root, Authoritative, Resolver, Cache) has a cryptographic identity based on an Ed25519 key pair.

### Key Setup

At initialisation, each node MUST generate an Ed25519 key pair and protect the private key using appropriate access controls. Key rotation MUST be supported without service interruption (see §17.5 for trust chain implications).

```
# Generate Ed25519 key pair for a registry node
openssl genpkey -algorithm ed25519 -out node-private.pem
openssl pkey -in node-private.pem -pubout -out node-public.pem
```

### Node Manifest

Every node MUST publish a manifest at `/.well-known/rcan-node.json`. This endpoint MUST be publicly accessible over HTTPS with no authentication required.

```json
{
  "node_id": "https://registry.boston-dynamics.com",
  "node_type": "authoritative",
  "namespace_prefix": "BD",
  "operator": "Boston Dynamics, Inc.",
  "rcan_version": "1.3",
  "public_key": "ed25519:MCowBQYDK2VwAyEA...",
  "public_key_fingerprint": "sha256:3a7f9c2d...",
  "sync_interval_seconds": 3600,
  "api_base": "https://registry.boston-dynamics.com/api/rcan/v1",
  "delegation_cert": {
    "namespace_prefix": "BD",
    "node_url": "https://registry.boston-dynamics.com",
    "node_pubkey": "ed25519:MCowBQYDK2VwAyEA...",
    "granted_at": "2026-01-01T00:00:00Z",
    "expires_at": "2027-01-01T00:00:00Z",
    "root_signature": "ed25519:rootsig456..."
  }
}
```

### Delegation Certificate

Authoritative nodes MUST obtain a delegation certificate from root before registering any RRNs under their delegated prefix. The delegation cert is a JSON document signed by root's Ed25519 private key.

```json
{
  "namespace_prefix": "BD",
  "node_url": "https://registry.boston-dynamics.com",
  "node_pubkey": "ed25519:MCowBQYDK2VwAyEA...",
  "granted_at": "2026-01-01T00:00:00Z",
  "expires_at": "2027-01-01T00:00:00Z",
  "root_signature": "ed25519:rootsig456..."
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `namespace_prefix` | MUST | The RRN prefix delegated to this node (e.g. `BD`) |
| `node_url` | MUST | Canonical URL of the Authoritative node |
| `node_pubkey` | MUST | Ed25519 public key of the node (`ed25519:<base64>`) |
| `granted_at` | MUST | ISO 8601 timestamp when delegation was granted |
| `expires_at` | MUST | ISO 8601 expiry; node MUST renew before this date |
| `root_signature` | MUST | Ed25519 signature over the other fields using root's private key |

!!! warning ""
    **Security note:** The delegation cert MUST be embedded in the node manifest at `/.well-known/rcan-node.json` and included in every sync message. Clients MUST verify the cert signature against root's public key before trusting any record from that node.

---

## 17.4 Sync Protocol

Authoritative nodes MUST synchronise their records to root (and optionally to other nodes) using a periodic pull-based mechanism with optional webhook push.

### Pull Sync (Required)

Nodes MUST poll their parent node every `sync_interval_seconds` (default: **3600 seconds**, minimum: **60 seconds**). Each poll fetches the delta of changed records since the last successful sync.

1. Node sends `GET {parent}/api/rcan/v1/sync?since={last_sync_iso}`
2. Parent responds with a JSON array of changed records (see §17.7 Wire Format)
3. Node applies changes, updating its local store
4. On conflict, root record MUST win (root is authoritative source of truth)
5. Node records the `synced_at` timestamp for the next poll cycle

### Webhook Push (Recommended)

Nodes SHOULD support webhook-based push from their parent. When a record changes at the parent, the parent SHOULD POST a sync payload to all registered webhook URLs of child nodes.

```
# Register webhook at parent
POST https://rcan.dev/api/rcan/v1/nodes/webhooks
{
  "node_url": "https://registry.example.com",
  "webhook_url": "https://registry.example.com/api/rcan/v1/webhook",
  "secret": "sha256:<hmac-secret>"
}
```

### Conflict Resolution

| Scenario | Resolution |
|----------|-----------|
| Same RRN exists at node and root with different values | Root record wins; node MUST overwrite local copy |
| Record exists at node but not at root | Node-originated record is preserved (node is authoritative for its prefix) |
| Root revokes a record registered by an Authoritative node | Root revocation MUST propagate; node MUST update attestation to `revoked` |

!!! info ""
    **Implementation note:** Nodes SHOULD use exponential backoff (starting at 60 seconds, maximum 1 hour) when sync attempts fail due to network errors. Persistent sync failures MUST be surfaced in the node's health endpoint.

---

## 17.5 Trust Chain

RCAN uses a two-level trust chain rooted at the root registry. All verification flows ultimately validate against root's Ed25519 public key.

```
Root (rcan.dev)
  Trust Anchor — Ed25519 public key published at /.well-known/rcan-node.json
        ↓ signs delegation cert
Authoritative Node
  Delegation cert signed by root · Signs robot records
        ↓ signs robot record
Robot Record (RRN)
  node_signature: ed25519:...
```

### Trust Rules

1. **Root's public key is the trust anchor.** It is published at `https://rcan.dev/.well-known/rcan-node.json`. Clients SHOULD pin this key and verify it out-of-band.
2. **Each Authoritative node holds a delegation cert** signed by root's Ed25519 private key. The cert binds the node's public key to its namespace prefix.
3. **Each robot record SHOULD carry the registering node's signature** over the record's canonical JSON, using the node's Ed25519 private key.
4. **Clients MUST verify** the following chain before trusting a record:
    - Record signature is valid against the node's public key
    - Node's delegation cert signature is valid against root's public key
    - Delegation cert has not expired
    - RRN prefix in the record matches the prefix in the delegation cert

### Key Rotation

Nodes MUST support key rotation without breaking existing records. When rotating:

1. Generate new Ed25519 key pair
2. Obtain updated delegation cert from root (Authoritative nodes)
3. Publish new `/.well-known/rcan-node.json` with new key and cert
4. Re-sign existing records with the new key (MAY be done lazily on next sync)
5. Keep old public key available for verifying pre-rotation records for at least 30 days

---

## 17.6 Resolution Algorithm

Clients and Resolver nodes MUST implement the following deterministic algorithm when resolving an RRN. The algorithm handles legacy RRNs, delegated RRNs, network failures, and cache fallback.

```python
function resolve(rrn):
  # Legacy RRN format: RRN-XXXXXXXX (8 hex chars, no prefix)
  if rrn matches /^RRN-[0-9A-F]{8}$/:
    return root.lookup(rrn)

  # Delegated RRN format: RRN-{PREFIX}-{SEQUENCE}
  prefix = extract_prefix(rrn)  // "RRN-BD-00000001" → "BD"
  node = root.delegations[prefix]
  if not node:
    return NOT_FOUND

  try:
    record = node.lookup(rrn)
    verify_signature(record, node.pubkey)
    verify_delegation(node.cert, root.pubkey)
    return record
  except NetworkError:
    return cache.lookup(rrn) or UNAVAILABLE
```

### Step-by-step

1. **Check cache.** If a valid (non-expired) cached record exists for the RRN, return it immediately without a network call.
2. **Legacy RRN check.** If the RRN matches the pattern `RRN-[0-9A-F]{8}` (no prefix), query root directly.
3. **Extract prefix.** From `RRN-BD-00000001`, extract `BD`. Look up `BD` in root's `namespace_delegations` table.
4. **Resolve at Authoritative node.** Query the Authoritative node for the full RRN. Verify both the record signature and the delegation cert.
5. **Network failure fallback.** If the Authoritative node is unreachable, attempt to serve from local cache (including stale entries, with appropriate error code `6006 CACHE_STALE`). If no cache entry exists, return `6005 NODE_UNAVAILABLE`.

!!! warning ""
    **Security requirement:** Clients MUST NOT skip signature verification even when serving from cache. A cached record with an invalid signature MUST be treated as a cache miss and a fresh lookup MUST be attempted.

---

## 17.7 Wire Format

All node-to-node sync messages use a standard JSON envelope. This format is used for both pull sync responses and webhook push payloads.

### Sync Message (Node → Node)

```json
{
  "protocol": "rcan-sync/1.0",
  "from_node": "https://registry.example.com",
  "to_node": "https://rcan.dev",
  "since": "2026-03-06T00:00:00Z",
  "records": [
    {
      "rrn": "RRN-BD-00000001",
      "robot_name": "Atlas Unit 001",
      "registered_at": "2026-01-15T09:00:00Z",
      "attestation": "active",
      "node_signature": "ed25519:abc123..."
    }
  ],
  "signature": "ed25519:xyz789..."
}
```

### Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `protocol` | string | MUST | Protocol identifier and version. Currently `"rcan-sync/1.0"`. |
| `from_node` | string (URL) | MUST | Canonical URL of the sending node. |
| `to_node` | string (URL) | MUST | Canonical URL of the receiving node. |
| `since` | string (ISO 8601) | MUST | Only records modified after this timestamp are included. |
| `records` | array | MUST | Array of robot record objects changed since `since`. May be empty. |
| `signature` | string | MUST | Ed25519 signature over the message (`ed25519:<base64>`), signed by the sending node's private key. |

### Record Object

Each element of the `records` array represents one robot registration. The minimum required fields are:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rrn` | string | MUST | Robot Registration Number (e.g. `RRN-BD-00000001`) |
| `robot_name` | string | MUST | Human-readable robot name |
| `registered_at` | ISO 8601 | MUST | When the robot was first registered at the Authoritative node |
| `attestation` | string (enum) | MUST | One of: `active`, `pending`, `suspended`, `revoked` |
| `node_signature` | string | SHOULD | Ed25519 signature over this record's canonical JSON, from the Authoritative node |

### Transport Requirements

- All sync messages MUST be transmitted over HTTPS (TLS 1.2 minimum, TLS 1.3 RECOMMENDED).
- Webhook deliveries MUST include an `X-RCAN-Signature: sha256=<hmac>` header for receiver validation.
- Receivers MUST validate the message-level `signature` field using the sender's published public key from their `/.well-known/rcan-node.json`.
- Receivers MUST reject messages where `to_node` does not match their own node URL.

---

## 17.8 Error Codes

The following error codes are specific to the Distributed Registry Node Protocol. They supplement the general RCAN error codes. These codes MUST be returned in the `code` field of an `ERROR` message or HTTP error response body.

| Code | Name | HTTP Status | Meaning |
|------|------|-------------|---------|
| `6001` | `NODE_NOT_FOUND` | 404 | No Authoritative node is registered for the RRN prefix. The prefix may not be delegated or may have been revoked. |
| `6002` | `DELEGATION_INVALID` | 403 | Node delegation certificate signature verification failed. The cert may be expired, tampered with, or signed by an unrecognised key. |
| `6003` | `RECORD_SIG_INVALID` | 403 | Robot record signature verification failed. The record may have been tampered with in transit or after registration. |
| `6004` | `SYNC_CONFLICT` | 409 | A sync conflict was detected (same RRN with differing values). Root record wins. The conflicting node record has been overwritten. |
| `6005` | `NODE_UNAVAILABLE` | 503 | The Authoritative node for this RRN prefix is currently unreachable. No cached record is available. |
| `6006` | `CACHE_STALE` | 206 | A cached record was found but its TTL has expired. A live fetch from the Authoritative node failed. The stale record is returned with this code to allow callers to decide how to proceed. |

### Error Response Format

```json
{"code": 6001, "name": "NODE_NOT_FOUND", "message": "No authoritative node registered for prefix 'XY'", "rrn": "RRN-XY-00000001"}
```

!!! info ""
    **Note on 6006 CACHE_STALE:** This is a partial success — the caller receives data but SHOULD treat it as unverified. The `stale_since` field (ISO 8601) SHOULD be included in the response to indicate when the TTL expired.
