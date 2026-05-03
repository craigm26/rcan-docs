# §1 Robot URI (RURI)

*Status: Stable · RCAN v2.1*

Every robot on the RCAN network has a globally unique Robot URI (RURI). Two syntactic forms are valid: a canonical form (for cloud/registry deployments) and a local shorthand (for LAN-only deployments). Parsers MUST support both forms.

---

## 1.1 Overview

The Robot URI (RURI) is the primary identifier for any RCAN-capable device. It uniquely identifies a robot across registries, networks, and time. RURIs appear in JWT `aud` claims, message envelopes, mDNS TXT records, and audit logs.

A RURI is always prefixed with the `rcan://` scheme. The authority section distinguishes canonical from shorthand form: canonical URIs contain exactly one forward slash after the authority (registry domain), while shorthand URIs use dot-separated segments and no registry.

---

## 1.2 Syntactic Forms

### Canonical form

Used when a registry domain is available (cloud or self-hosted):

```
rcan://<registry>/<manufacturer>/<model>/<device-id>[:<port>][/<capability>]
```

### Local shorthand

When no registry domain is needed (LAN-only deployments), the authority section uses dot-separated segments. Parsers MUST expand this to the canonical form with `local.rcan` as the registry:

```
rcan://<manufacturer>.<model>.<instance>[/<capability>]
```

Example: `rcan://opencastor.rover.abc123` expands to `rcan://local.rcan/opencastor/rover/abc123`

```bash
# Canonical form (with registry)
rcan://continuon.cloud/continuon/companion-v1/d3a4b5c6
rcan://continuon.cloud/continuon/companion-v1/d3a4b5c6/arm
rcan://my-server.lan/acme/bot-x1/a1b2c3d4:9000/teleop

# Local shorthand (no registry; implies local.rcan)
rcan://acme.bot-x1.a1b2c3d4        # → rcan://local.rcan/acme/bot-x1/a1b2c3d4
rcan://opencastor.rover.abc123/nav  # → rcan://local.rcan/opencastor/rover/abc123/nav

# Special addresses
rcan://local.rcan/discovered/192.168.1.42:8080  # mDNS-discovered device

# ── v2.1: Signed RURI (required at L2+) ──────────────────────
# ?sig=<base64url> carries a detached Ed25519 signature over the RURI path
rcan://registry.rcan.dev/manufacturer/model/version/device-id?sig=<base64url>

# The signature covers the path only (not the query string):
# sign( "registry.rcan.dev/manufacturer/model/version/device-id" )
```

---

## 1.3 Components

| Component | Description | Required |
|-----------|-------------|----------|
| `registry` | Root registry domain (`continuon.cloud`, `local.rcan`) | Canonical only |
| `manufacturer` | Manufacturer or organisation namespace | Yes |
| `model` | Robot model identifier | Yes |
| `device-id` | UUID or 8-char hex short-form; dot-form uses any alphanumeric slug | Yes |
| `port` | Communication port (default: 8000) | No |
| `capability` | Specific endpoint path (e.g. `/arm`, `/nav`) | No |

---

## 1.4 Validation Patterns

Implementations MUST use the following regex patterns to validate RURIs before processing:

```text
# Canonical RURI
^rcan://([a-z0-9][a-z0-9.-]*[a-z0-9])/([a-z0-9][a-z0-9-]*[a-z0-9])/([a-z0-9][a-z0-9-]*[a-z0-9])/([0-9a-f]{8}(?:-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})?)(?::(\d{1,5}))?(/[a-z][a-z0-9/-]*)?$

# Local shorthand  (authority contains dots, no path segments for identity)
^rcan://([a-z0-9][a-z0-9-]*)\.([a-z0-9][a-z0-9-]*)\.([a-z0-9]{4,36})(/[a-z][a-z0-9/-]*)?$
```

!!! note ""
    The device-id segment accepts either a full UUID v4 (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) or an 8-character lowercase hex short-form. The dot-form shorthand accepts any alphanumeric slug of 4–36 characters.

---

## 1.5 Wildcard Matching

The `*` character MAY appear in any segment for **audience matching in JWT tokens**. Wildcards are only valid in JWT `aud` claims — not in operational message routing.

Examples:

- `rcan://continuon.cloud/continuon/companion-v1/*` — grants access to all devices of that model
- `rcan://continuon.cloud/continuon/*/d3a4b5c6` — grants access to a specific device across all models of that manufacturer
- `rcan://local.rcan/*/*/abc123` — grants access to a specific device on any local registry

!!! info ""
    **Cross-reference:** Wildcard matching is used in [§5 Authentication](section-5.md) to validate the `aud` claim of JWT tokens. See also [§2](section-2.md) for fleet-scoped tokens.

---

## 1.6 Signed RURI (v2.1)

v2.1 introduces a signed RURI format required at **L2 conformance and above**. The `?sig=` query parameter carries a detached Ed25519 signature over the RURI path, enabling offline identity verification without a registry roundtrip.

### 1.6.1 Format

```
rcan://registry.rcan.dev/manufacturer/model/version/device-id?sig=<base64url>
```

The signature is computed over the **path only** — not the query string. This ensures the signature remains stable even if additional query parameters are added in future versions.

```bash
# Signing a RURI (pseudocode)
path  = uri.split("?")[0].replace("rcan://", "")
sig   = ed25519_sign(private_key, path.encode("utf-8"))
signed_ruri = uri + "?sig=" + base64url_encode(sig)

# Verifying
path  = ruri.split("?")[0].replace("rcan://", "")
sig   = base64url_decode(ruri.query_params["sig"])
valid = ed25519_verify(manufacturer_public_key, path.encode("utf-8"), sig)
# On failure: emit FAULT_REPORT { fault_code: "RURI_SIGNATURE_INVALID" }
```

### 1.6.2 Key Registration

The Ed25519 public key used to sign RURIs MUST be registered in the Robot Registry Foundation (RRF) against the manufacturer namespace. Receivers verify the signature using the manufacturer's registered key.

- L1 conformance: unsigned RURI is permitted
- L2+ conformance: signed RURI is REQUIRED. Unsigned RURIs MUST be rejected at L2+.
- Invalid signature: MUST emit `FAULT_REPORT (26)` with `fault_code: "RURI_SIGNATURE_INVALID"`

### 1.6.3 Relationship to firmware_hash

The signed RURI establishes *who* is sending; the `firmware_hash` envelope field (§3, field 13) establishes *what software* they are running. Together they form the v2.1 sender identity proof: verified origin + verified build.

### 1.6.4 PQC Hybrid Signing (v2.3)

When a node advertises `crypto_profile: "pqc-hybrid-v1"` in its `/.well-known/rcan-node.json`, the `?sig=` parameter MUST use the combined Ed25519 + ML-DSA-65 encoding:

```
rcan://registry.rcan.dev/manufacturer/model/version/device-id?sig=pqc-hybrid-v1.<ed25519_b64url>.<ml_dsa_b64url>
```

- Both halves are REQUIRED; a signature missing either half MUST be rejected.
- Receivers MUST verify both halves independently; either failure rejects the message.
- Legacy Ed25519-only (`?sig=<bare_b64url>`) is still accepted by receivers that do not require `pqc-hybrid-v1`.
- On failure: emit `FAULT_REPORT (26)` with `fault_code: "RURI_SIGNATURE_INVALID"`

```python
# Signing a RURI with pqc-hybrid-v1 (pseudocode, v2.3+)
path     = ruri.split("?")[0].replace("rcan://", "")
ed_sig   = ed25519_sign(ed25519_private_key, path.encode("utf-8"))
pq_sig   = ml_dsa_65_sign(ml_dsa_private_key, path.encode("utf-8"))
combined = "pqc-hybrid-v1." + base64url(ed_sig) + "." + base64url(pq_sig)
signed_ruri = ruri + "?sig=" + combined

# Verifying (v2.3+ receiver)
parts = sig.split(".")
# parts[0]=="pqc-hybrid-v1", parts[1]==ed25519 half, parts[2]==ml-dsa half
assert len(parts) == 3, "Not a pqc-hybrid-v1 sig"
ed_ok = ed25519_verify(ed25519_pubkey, path.encode("utf-8"), base64url_decode(parts[1]))
pq_ok = ml_dsa_65_verify(pqc_pubkey,   path.encode("utf-8"), base64url_decode(parts[2]))
# BOTH must pass — single-half sigs MUST be rejected
assert ed_ok and pq_ok, "RURI_SIGNATURE_INVALID"
```

See [Cryptographic Profiles](https://rcan.dev/docs/crypto/pqc-profile) for key sizes, migration timeline, and reference implementations.
