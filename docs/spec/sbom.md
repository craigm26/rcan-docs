# Supply Chain Attestation (SBOM)

*Status: Stable · RCAN v2.1 · MessageType 44 — SBOM_UPDATE*

!!! info ""
    **Overview:** Every RCAN v2.1 robot MUST publish a Software Bill of Materials (SBOM) at a well-known endpoint following CycloneDX v1.5+ format with RCAN-specific extensions. The `attestation_ref` field in every v2.1 message envelope (§3, field 14) points to this endpoint, enabling any message recipient to verify the sender's full software supply chain.

---

## Well-Known Endpoint

```
GET {ruri}/.well-known/rcan-sbom.json
```

Returns `Content-Type: application/json`. MUST be regenerated when any component version changes. The `attestation_ref` envelope field (§3, field 14) should point to this URL or to the RRF-hosted countersigned version.

---

## SBOM Schema + RCAN Extensions

RCAN SBOMs use [CycloneDX v1.5+](https://cyclonedx.org/) with a required `x-rcan-extensions` object:

```json
{
  "bomFormat":   "CycloneDX",
  "specVersion": "1.5",
  "version":     1,
  "metadata": {
    "timestamp": "2026-04-01T00:00:00Z",
    "component": {
      "type": "device",
      "name": "opencastor-rpi5-bob",
      "version": "v2026.4.1.0"
    }
  },
  "components": [
    {
      "type":    "library",
      "name":    "opencastor",
      "version": "2026.4.1.0",
      "hashes":  [{ "alg": "SHA-256", "content": "a1b2c3d4..." }],
      "purl":    "pkg:pypi/opencastor@2026.4.1.0"
    },
    {
      "type":    "library",
      "name":    "rcan",
      "version": "1.0.0",
      "hashes":  [{ "alg": "SHA-256", "content": "e5f6a7b8..." }],
      "purl":    "pkg:pypi/rcan@1.0.0"
    }
  ],
  "x-rcan-extensions": {
    "rrn":                   "RRN-000000000001",
    "firmware_hash":         "sha256:a1b2c3d4...",
    "attestation_signed_at": "2026-04-01T00:00:00Z",
    "rrf_countersignature":  "base64url..."
  }
}
```

### RCAN Extension Fields

| Extension Field | Required | Description |
|-----------------|----------|-------------|
| `rrn` | Yes | Robot Registration Number — links SBOM to RRF robot record |
| `firmware_hash` | Yes | Links SBOM to firmware manifest `build_hash` (see [Firmware Manifests](firmware.md)) |
| `attestation_signed_at` | Yes | UTC timestamp when RRF countersignature was issued |
| `rrf_countersignature` | L5 only | Ed25519 signature by RRF root key over the SBOM hash |

---

## SBOM_UPDATE (44)

Robots push their SBOM to the RRF registry using the `SBOM_UPDATE` message type:

```json
{
  "version":        "2.0.0",
  "message_id":     "uuid-v4",
  "source_ruri":    "rcan://rrf.rcan.dev/opencastor/rpi5-hailo/bob-001?sig=<base64url>",
  "target_ruri":    "rcan://rrf.rcan.dev/rrf/registry/v2/sbom",
  "type":           44,
  "firmware_hash":  "sha256:a1b2c3d4...",
  "attestation_ref":"https://rrf.rcan.dev/robots/RRN-000000000001/sbom",
  "payload": {
    "rrn":              "RRN-000000000001",
    "cyclonedx_version":"1.5",
    "components":       [],
    "x_rcan_firmware_hash":         "sha256:a1b2c3d4...",
    "x_rcan_attestation_signed_at": "2026-04-01T00:00:00Z"
  }
}
```

Note that the message itself carries `firmware_hash` and `attestation_ref` in its own envelope — bootstrapping the chain of trust from the very message that establishes it.

---

## RRF Countersigning

When a robot publishes an SBOM via `SBOM_UPDATE (44)`, the RRF:

1. Validates CycloneDX format and RCAN extension fields
2. Verifies `x-rcan-firmware-hash` matches a previously attested firmware manifest
3. Signs the SBOM hash with the RRF root Ed25519 key
4. Stores the countersigned SBOM at `/v2/robots/{rrn}/sbom`
5. Sets `rrf_countersignature` in the extension block

The countersigned URL should be used as the `attestation_ref` in subsequent message envelopes, replacing the local well-known endpoint once RRF countersigning is complete.

---

## Conformance Requirements

- **L2+ (required):** Robot MUST serve a valid SBOM at the well-known endpoint.
- **L2+ (required):** Every outgoing message MUST include `attestation_ref` in the envelope (§3, field 14) pointing to a reachable SBOM.
- **L2+ (required):** SBOM MUST include all three required RCAN extension fields (`rrn`, `firmware_hash`, `attestation_signed_at`).
- **L5 (required):** SBOM MUST be published to RRF via `SBOM_UPDATE (44)`. RRF countersignature MUST be present in `rrf_countersignature`.
- **All levels:** SBOM MUST be refreshed when any component version changes.
