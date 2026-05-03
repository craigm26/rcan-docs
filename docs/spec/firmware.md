# Firmware Manifests

*Status: Stable · RCAN v2.1 · MessageType 43 — FIRMWARE_ATTESTATION*

!!! info ""
    **Overview:** Every RCAN v2.1 robot MUST publish a signed firmware manifest at a well-known endpoint. The manifest proves what software is running on the robot and is signed by the manufacturer's key registered in the Robot Registry Foundation (RRF). The `firmware_hash` field in every v2.1 message envelope (§3, field 13) references this manifest.

---

## Well-Known Endpoint

Every RCAN v2.1 robot MUST serve its firmware manifest at:

```
GET {ruri}/.well-known/rcan-firmware-manifest.json
```

The endpoint MUST return `Content-Type: application/json`. The manifest MUST be regenerated and re-signed on every firmware update.

---

## Manifest Schema

```json
{
  "rrn":              "RRN-000000000001",
  "firmware_version": "v2026.4.1.0",
  "build_hash":       "sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "components": [
    { "name": "brain-runtime",  "version": "3.2.0", "hash": "sha256:e5f6a7b8..." },
    { "name": "hailo-firmware", "version": "4.17.0","hash": "sha256:c9d0e1f2..." },
    { "name": "rcan-py",        "version": "1.0.0", "hash": "sha256:a3b4c5d6..." }
  ],
  "signed_at": "2026-04-01T00:00:00Z",
  "signature":  "base64url-encoded-ed25519-sig-over-canonical-json"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rrn` | string | Yes | Robot Registration Number |
| `firmware_version` | string | Yes | Semver or CalVer version string |
| `build_hash` | string | Yes | SHA-256 of the full firmware bundle, prefixed `"sha256:"` |
| `components` | array | Yes | Per-component name/version/hash records |
| `signed_at` | ISO 8601 | Yes | UTC timestamp when manifest was signed |
| `signature` | string | Yes | Ed25519 signature over canonical JSON (all fields except signature), base64url-encoded |

---

## Signing Requirements

- The signing key MUST be the manufacturer's Ed25519 private key registered in RRF.
- The signature input is the **canonical JSON** of the manifest object with the `signature` field omitted, keys sorted lexicographically, no trailing whitespace.
- The manifest MUST be re-signed after any change to `firmware_version`, `build_hash`, or any `components` entry.
- Manifests older than 90 days SHOULD be refreshed even if firmware has not changed.

---

## Verification Flow

On first connection and after any firmware update, a v2.1 L2+ receiver MUST verify the peer's manifest:

```python
# Verify a received firmware manifest
path     = ruri.path_only()                  # strip ?sig= query param
sig_bytes= base64url_decode(manifest["signature"])
pub_key  = rrf.get_manufacturer_key(manifest["rrn"])
valid    = ed25519_verify(pub_key, canonical_json(manifest_without_sig), sig_bytes)

if not valid:
    emit FAULT_REPORT { fault_code: "FIRMWARE_INTEGRITY_FAILURE", severity: "critical" }
    halt_new_connections()
```

!!! danger ""
    **Conformance requirement:** A failed signature check MUST emit `FAULT_REPORT (26)` with `fault_code: "FIRMWARE_INTEGRITY_FAILURE"` and MUST halt acceptance of new connections from that peer until the manifest is corrected.

---

## FIRMWARE_ATTESTATION (43)

Robots publish their manifest to the RRF registry using the `FIRMWARE_ATTESTATION` message type. This allows RRF to countersign manifests and display attestation status on robot detail pages.

```json
{
  "version":        "2.0.0",
  "message_id":     "uuid-v4",
  "source_ruri":    "rcan://rrf.rcan.dev/opencastor/rpi5-hailo/bob-001?sig=<base64url>",
  "target_ruri":    "rcan://rrf.rcan.dev/rrf/registry/v2/attestation",
  "type":           43,
  "firmware_hash":  "sha256:a1b2c3d4...",
  "attestation_ref":"https://rrf.rcan.dev/robots/RRN-000000000001/sbom",
  "payload": {
    "rrn":              "RRN-000000000001",
    "firmware_version": "v2026.4.1.0",
    "build_hash":       "sha256:a1b2c3d4...",
    "components":       ["..."],
    "signed_at":        "2026-04-01T00:00:00Z",
    "signature":        "base64url..."
  }
}
```

The RRF registry validates the manufacturer signature before storing. On success, RRF adds its own countersignature and makes the manifest available at `GET /v2/robots/{rrn}/firmware-manifest`.

---

## Conformance Requirements

- **L2+ (required):** Robot MUST serve a valid, signed manifest at the well-known endpoint.
- **L2+ (required):** Every outgoing message MUST include `firmware_hash` in the envelope (§3, field 13) equal to `build_hash` in the manifest.
- **L2+ (required):** On first connection, the peer's manifest signature MUST be verified. Failed verification triggers `FIRMWARE_INTEGRITY_FAILURE` fault.
- **L5 (required):** Manifest MUST be published to RRF via `FIRMWARE_ATTESTATION (43)`. RRF countersignature MUST be present.
- **All levels:** Manifests MUST be refreshed on every firmware update and SHOULD be refreshed every 90 days.
