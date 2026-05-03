# §21 Robot Registry Integration

*Status: Stable · New in v1.3 · RCAN v1.3*

!!! info ""
    **Overview:** §21 specifies how a RCAN-compliant robot runtime integrates with an external Robot Registry (such as the [Robot Registry Foundation](https://robotregistryfoundation.org)). It defines the canonical relationship between Robot Registration Numbers (RRNs) and Robot URIs (RURIs), the challenge-response ownership proof flow, and two new RCAN message types: `REGISTRY_REGISTER` and `REGISTRY_RESOLVE`.

---

## 21.1 Overview

Robot runtimes implementing RCAN operate under two independent identity systems:

- **RURI** (`rcan://rcan.dev/continuonai/opencastor-rpi5/bob`) — the protocol-level address used in every RCAN message (§2). Assigned by the runtime operator; unique within the registry host's namespace.
- **RRN** (`RRN-000000000042`) — a globally unique, registry-assigned opaque identifier. Issued by the Robot Registry after registration. Stable across network changes, renames, and ownership transfers.

§21 defines:

- How a RURI and an RRN are canonically linked in the registry
- How a runtime proves it controls a RURI (ownership proof via Ed25519 challenge-response)
- The `REGISTRY_REGISTER` and `REGISTRY_RESOLVE` RCAN message types
- How verification tier upgrades flow through the registry

---

## 21.2 RRN ↔ RURI Mapping

An RRN (`RRN-000000000001`) is a globally unique, registry-assigned opaque identifier. A RURI (`rcan://rcan.dev/continuonai/opencastor-rpi5/bob`) is the RCAN address used at the protocol level.

### Canonical Association

A robot MAY associate one or more RURIs with its RRN. The association is stored in the registry and discoverable via the registry resolution API. The primary RURI is the address currently used in RCAN messages.

```bash
# RURI → RRN
GET /api/v1/resolve?ruri=rcan://rcan.dev/continuonai/opencastor-rpi5/bob
→ {"rrn": "RRN-000000000042", "status": "active", "verification_tier": "verified"}

# RRN → RURI
GET /api/v1/robots/RRN-000000000042
→ {"rrn": "RRN-000000000042", "ruri": "rcan://rcan.dev/continuonai/opencastor-rpi5/bob", ...}
```

| Direction | Endpoint | Response field |
|-----------|----------|---------------|
| RURI → RRN | `GET /api/v1/resolve?ruri=rcan://...` | `rrn` |
| RRN → RURI | `GET /api/v1/robots/{rrn}` | `ruri` |

!!! warning ""
    **Note:** A robot that changes its RURI (e.g. after a registry migration) MUST update the mapping at the registry. The RRN remains stable. Resolvers SHOULD cache mappings with the TTL returned by the registry.

### 21.2.1 Structured URI RRN Format

In addition to the numeric format (`RRN-000000000042`), §21.2.1 defines an optional **structured URI RRN** format that encodes the entity type and organisational namespace directly in the identifier. This enables registries to track not just whole robots, but also individual *components*, *sensors*, and modular *assemblies*.

```bash
# Structured URI RRN — 4-segment (recommended)
rrn://[org]/[category]/[model]/[id]
  → rrn://opencastor.com/robot/v2/unit-001
  → rrn://opencastor.com/component/hailo8/module-42
  → rrn://luxonis.com/sensor/oak-d/cam-007
  → rrn://opencastor.com/assembly/perception-stack/asm-003

# 3-segment (category + id, model omitted)
rrn://[org]/[category]/[id]
  → rrn://opencastor.com/robot/bob

# 2-segment legacy (category=robot assumed)
rrn://[org]/[id]
  → rrn://example.org/rover-1

# Numeric (registry-assigned, always valid)
RRN-000000000042
RRN-BD-000000000042   # with manufacturer prefix
```

#### Entity Categories

| Category | Description | Has RURI? | Example |
|----------|-------------|-----------|---------|
| `robot` | Fully assembled robot unit. First-class registry citizen. | Required | `rrn://opencastor.com/robot/v2/unit-001` |
| `component` | Individual hardware piece (NPU, motor controller, etc.). May lack network identity. | Optional | `rrn://opencastor.com/component/hailo8/module-42` |
| `sensor` | Passive sensing device (camera, LiDAR, IMU). Registered for telemetry correlation. | Optional | `rrn://luxonis.com/sensor/oak-d/cam-007` |
| `assembly` | Modular subsystem (perception stack, arm+camera+NPU). Swappable between robot bodies. | Optional | `rrn://opencastor.com/assembly/perception-stack/asm-003` |

#### Format Rules

- MUST start with `rrn://`
- MUST contain 2–4 slash-separated path segments after the scheme
- Segment 0 (org): non-empty, alphanumerics + `-._`
- Segment 1 (category, when 3–4 segments): MUST be one of `robot`, `component`, `sensor`, `assembly`
- Segment 2 (model, 4-segment only): non-empty
- Segment 3 / last segment (id): non-empty, unique within scope
- 2-segment RRNs (`rrn://org/id`) are the legacy flat format; implementors SHOULD treat category as `robot`

#### Metadata Block

The `REGISTRY_REGISTER` payload's `metadata` object is extended with component-tracking fields:

| Field | Type | Description |
|-------|------|-------------|
| `serial` | string | Manufacturer serial number |
| `firmware` | string | Firmware / software version string |
| `category` | string | Explicit category override (redundant if encoded in URI RRN) |
| `components` | string[] | Child RRNs — for assemblies and robots listing their parts |
| `parent_rrn` | string | Parent RRN — for components/sensors belonging to a robot or assembly |

!!! info ""
    **Coexistence:** Numeric RRNs (`RRN-NNNNNNNNNNNN`) and structured URI RRNs (`rrn://...`) are both valid in §21. Registries MUST accept either form. The numeric format is appropriate for central registry assignment; the URI format is better suited for distributed registries and self-describing embedded systems that announce their own RRN at boot.

### 21.2.2 Multi-Type Entity Numbering (v2.2)

RCAN v2.2 extends the registry to track four distinct entity types, each with its own prefixed sequential identifier. All identifiers share the same 12-digit zero-padded format and are globally unique within the issuing registry.

| Prefix | Entity Type | Description | Example |
|--------|-------------|-------------|---------|
| `RRN` | **Whole Robot** | A fully assembled robot unit with its own identity, key pair, and RURI. | `RRN-000000000001` |
| `RCN` | **Component** | Individual hardware piece (NPU, camera, sensor, motor controller). Linked to a parent RRN. | `RCN-000000000001` |
| `RMN` | **AI Model** | A registered AI model (VLA, LLM, world model, embedding model, etc.) with provenance and license metadata. | `RMN-000000000001` |
| `RHN` | **AI Harness** | A registered agent/harness definition — the pipeline that orchestrates models for a robot task. References RMN IDs. | `RHN-000000000001` |

```bash
# RCAN v2.2 — Multi-Type Entity Identifiers
# All prefixes follow the same 12-digit zero-padded sequential format.
# IDs are registry-assigned; they are globally unique within the issuing registry.

RRN-000000000001   # Robot Registration Number     — whole robot assembly
RCN-000000000001   # Robot Component Number        — hardware component (NPU, camera, etc.)
RMN-000000000001   # Robot Model Number            — AI model (VLA, LLM, JEPA, embedding…)
RHN-000000000001   # Robot Harness Number          — AI harness / agent definition

# Format rule:  {PREFIX}-{000000000001..999999999999}
# Prefix chars: RRN | RCN | RMN | RHN (case-sensitive, always uppercase)
# Sequence:     12-digit zero-padded decimal, monotonically increasing per type
```

!!! warning ""
    **Provenance chain:** A complete robot deployment can be described as: `RHN → [RMN, RMN, …]` (harness uses models) + `RRN → [RCN, RCN, …]` (robot has components). EU AI Act Art. 11 technical documentation SHOULD include all four IDs for each deployment.

---

## 21.3 Ownership Proof

A runtime proves it controls a RURI by signing a challenge issued by the registry using its Ed25519 private key — the same key used to sign RCAN messages per §5.

```bash
# Step 1 — Request challenge
POST /api/v1/challenge
{"ruri": "rcan://rcan.dev/continuonai/opencastor-rpi5/bob"}

# Response
{"challenge": "a3f9c2d18b4e7f60...", "expires_at": "2026-03-12T12:00:00Z"}

# Step 2 — Sign challenge with Ed25519 private key (same key used for §5 message signing)
signature = ed25519_sign(private_key, challenge_hex)

# Step 3 — Submit proof
POST /api/v1/verify
{
  "ruri": "rcan://rcan.dev/continuonai/opencastor-rpi5/bob",
  "challenge": "a3f9c2d18b4e7f60...",
  "signature": "<base64url>",
  "public_key": "<base64url>"
}

# Response — robot promoted to "verified" tier
{"status": "verified", "rrn": "RRN-000000000042", "verification_tier": "verified"}
```

| Verification Tier | Requirement | Trust Level |
|------------------|-------------|-------------|
| `community` | Self-reported registration, no proof required | Low — unverified |
| `verified` | Ownership proof via Ed25519 challenge-response | Medium — cryptographically proven |
| `manufacturer` | Verified + manufacturer delegation cert (§17) | High — manufacturer-attested |
| `certified` | Third-party certification (out of scope for §21) | Highest |

!!! info ""
    **Security note:** The challenge nonce MUST be at least 32 bytes of cryptographically random data. The challenge MUST expire within 5 minutes (`expires_at`). Registries MUST reject replayed challenges. A challenge is single-use.

---

## 21.4 REGISTRY_REGISTER Message

A new RCAN message type that a runtime sends to a registry node to register or update its RURI↔RRN association and public key. The registry node processes the message and responds with a `REGISTRY_REGISTER_RESULT`.

### Request

```json
{
  "type": "REGISTRY_REGISTER",
  "payload": {
    "rrn": "RRN-000000000042",
    "ruri": "rcan://rcan.dev/continuonai/opencastor-rpi5/bob",
    "public_key": "MCowBQYDK2VwAyEA...",
    "metadata": {
      "name": "Bob",
      "manufacturer": "ContinuonAI",
      "model": "opencastor-rpi5",
      "serial": "OC2-2026-001",
      "firmware": "v2026.3.13.10"
    }
  }
}
```

### Response

```json
{
  "type": "REGISTRY_REGISTER_RESULT",
  "payload": {
    "rrn": "RRN-000000000042",
    "status": "registered",
    "verification_tier": "community"
  }
}
```

### Request Payload Fields

| Field | Required | Description |
|-------|----------|-------------|
| `rrn` | OPTIONAL | RRN previously assigned by the registry. Omit on first registration; the registry will assign one. |
| `ruri` | MUST | The robot's RCAN URI. Must match the `source_ruri` in the outer RCAN message envelope. |
| `public_key` | MUST | Ed25519 public key, base64url-encoded. Used to verify ownership proofs and future message signatures. |
| `metadata.name` | OPTIONAL | Human-readable robot name. |
| `metadata.manufacturer` | OPTIONAL | Manufacturer name. |
| `metadata.model` | OPTIONAL | Model identifier. |

### Result Fields

| Result field | Type | Description |
|-------------|------|-------------|
| `rrn` | string | Assigned (new registration) or confirmed (existing) RRN. |
| `status` | `"pending" \| "registered" \| "verified"` | Current registration status. |
| `verification_tier` | `"community" \| "verified" \| "manufacturer" \| "certified"` | Current verification tier. |

---

## 21.5 REGISTRY_RESOLVE Message

A RCAN message type for resolving a RURI or RRN to its full registry record. Useful for runtime-to-runtime lookups and fleet management systems.

### Request

```json
{
  "type": "REGISTRY_RESOLVE",
  "payload": {
    "query": "RRN-000000000042"
  }
}
```

### Response

```json
{
  "type": "REGISTRY_RESOLVE_RESULT",
  "payload": {
    "rrn": "RRN-000000000042",
    "ruri": "rcan://rcan.dev/continuonai/opencastor-rpi5/bob",
    "status": "active",
    "verification_tier": "verified",
    "public_key": "MCowBQYDK2VwAyEA..."
  }
}
```

### Result Fields

| Result field | Type | Description |
|-------------|------|-------------|
| `rrn` | string | Resolved Robot Registration Number. |
| `ruri` | string | Current primary RURI for this robot. |
| `status` | `"active" \| "inactive" \| "unknown"` | Current registry status of this robot. |
| `verification_tier` | string | Current verification tier (see §21.3). |
| `public_key` | string | Ed25519 public key. Present only for `verified` tier or above. MUST NOT be returned for `community` tier without explicit opt-in. |

!!! warning ""
    **Privacy note:** Registry implementors MUST NOT return `public_key` for unverified (`community` tier) robots unless the robot owner has explicitly enabled public key disclosure. Verified tier robots implicitly consent by completing the ownership proof flow.

---

## 21.6 Conformance Level L4: Registry Integration

A runtime achieves **L4 (Registry Integration)** conformance by satisfying all L1–L3 requirements plus the following:

**Required:**

- Implements `REGISTRY_REGISTER` message type
- Implements `REGISTRY_RESOLVE` message type
- Signs registry challenges with the same Ed25519 key used for RCAN message signing (§5)
- Stores the assigned RRN in robot config (`rcan_protocol.rrn`)
- Includes RRN as optional field in `CONNECT` messages

**Recommended:**

- Completes ownership proof flow on first registration
- Caches REGISTRY_RESOLVE results with registry-provided TTL
- Refreshes RRN mapping on RURI change
- Exposes RRN in discovery responses (§4)

### Config Schema Extension

Runtimes claiming L4 conformance MUST support the following `rcan.yaml` fields:

```yaml
rcan_version: "1.3.0"

metadata:
  robot_name: "Bob"
  robot_uuid: "550e8400-e29b-41d4-a716-446655440000"

registry:
  # Numeric RRN (registry-assigned, legacy format still supported)
  rrn: "RRN-000000000042"
  # OR structured URI RRN (recommended for new deployments)
  # rrn: "rrn://opencastor.com/robot/v2/unit-001"
  registry_url: "https://robotregistryfoundation.org/api/v1"
```

| Field | Required for L4 | Description |
|-------|----------------|-------------|
| `registry.rrn` | MUST (after registration) | Robot Registration Number assigned by the registry |
| `registry.registry_url` | OPTIONAL | Registry API base URL. Defaults to `https://robotregistryfoundation.org/api/v1` |

---

## 21.7 Implementation Notes

### Optionality

L4 conformance is **OPTIONAL** — robots can implement RCAN (L1–L3) without registry integration. §21 applies only to runtimes that choose to integrate with an external registry for global identity and discoverability.

### Default Registry

Unless overridden by `registry.registry_url` in config, implementations MUST use `https://robotregistryfoundation.org/api/v1` as the default registry endpoint.

### Key Re-use

The same Ed25519 key pair used for RCAN message signing (§5) is intentionally re-used for registry ownership proofs. This eliminates key management complexity and creates a single cryptographic identity for the robot: the same key that authorises commands also proves registry ownership.

!!! warning ""
    **Key rotation:** When a runtime rotates its Ed25519 key (§5), it MUST re-run the ownership proof flow against the registry with the new public key before the old key is retired. Failure to do so will downgrade the robot from `verified` to `community` tier.
