# Authority Access

*Status: Stable · RCAN v2.1 · MessageTypes 41–42*

!!! danger ""
    **Regulatory deadline:** EU AI Act Art. 16 applies to providers of high-risk AI systems from **August 2, 2026**. Robots using RCAN v2.1 with AUTHORITY_ACCESS handlers registered satisfy the Art. 16(j) cooperation requirement at the protocol level.

---

## Overview

The authority access protocol enables designated regulatory authorities to request structured audit data from RCAN v2.1 robots. It provides a standardized, verifiable interface for the cooperation requirement under EU AI Act Article 16(j) and similar regulatory frameworks.

Authority requests are authenticated using the same RURI signing mechanism as robot-to-robot communication. An authority MUST have a RURI registered in RRF with an `authority`-scoped token to issue valid requests.

---

## EU AI Act Article 16 Mapping

| Article | Requirement | RCAN v2.1 Coverage |
|---------|-------------|-------------------|
| Art. 16(a) | Quality management system | SBOM + firmware manifest provide continuous quality evidence |
| Art. 16(d) | Technical documentation | Firmware version + component list always current; TRANSPARENCY records (type 16) |
| Art. 12 | Logging capability | Commitment chain (§16) satisfies automatic logging requirement |
| Art. 16(j) | Authority cooperation | AUTHORITY_ACCESS (41) + AUTHORITY_RESPONSE (42) provide structured audit data access |
| Art. 61 | Post-market monitoring | Continuous SBOM + firmware attestation feed post-market monitoring evidence |

---

## AUTHORITY_ACCESS (41)

Sent by a regulatory authority to a robot requesting access to audit data.

```json
{
  "version":        "2.0.0",
  "message_id":     "uuid-v4",
  "source_ruri":    "rcan://authority.eu-ai-act.europa.eu/regulator/nca/v2",
  "target_ruri":    "rcan://rrf.rcan.dev/opencastor/rpi5-hailo/bob-001?sig=<base64url>",
  "type":           41,
  "firmware_hash":  "sha256:regulator-build-hash...",
  "attestation_ref":"https://authority.eu-ai-act.europa.eu/sbom/regulator-v2",
  "payload": {
    "request_id":     "audit-2026-08-01-001",
    "authority_id":   "EU-AI-ACT-NCA-DE",
    "requested_data": ["audit_chain", "transparency_records", "sbom", "firmware_manifest"],
    "justification":  "Routine post-market monitoring per EU AI Act Art. 61",
    "expires_at":     1754006400
  }
}
```

### Payload Fields

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | string | Unique request identifier (correlated in response) |
| `authority_id` | string | Authority identifier (e.g. `"EU-AI-ACT-NCA-DE"`) |
| `requested_data` | string[] | Allowed values: `"audit_chain"`, `"transparency_records"`, `"sbom"`, `"firmware_manifest"` |
| `justification` | string | Human-readable reason for the request |
| `expires_at` | int64 | Unix timestamp — request must be responded to before this time |

---

## AUTHORITY_RESPONSE (42)

Sent by the robot in reply to an AUTHORITY_ACCESS request.

```json
{
  "version":        "2.0.0",
  "message_id":     "uuid-v4",
  "source_ruri":    "rcan://rrf.rcan.dev/opencastor/rpi5-hailo/bob-001?sig=<base64url>",
  "target_ruri":    "rcan://authority.eu-ai-act.europa.eu/regulator/nca/v2",
  "type":           42,
  "firmware_hash":  "sha256:a1b2c3d4...",
  "attestation_ref":"https://rrf.rcan.dev/robots/RRN-000000000001/sbom",
  "payload": {
    "request_id":   "audit-2026-08-01-001",
    "rrn":          "RRN-000000000001",
    "provided_at":  1753920000,
    "data": {
      "audit_chain":            [],
      "transparency_records":   [],
      "sbom_url":               "https://rrf.rcan.dev/robots/RRN-000000000001/sbom",
      "firmware_manifest_url":  "https://rrf.rcan.dev/robots/RRN-000000000001/firmware-manifest"
    }
  }
}
```

---

## Request Flow

1. Authority issues `AUTHORITY_ACCESS (41)` to the robot's RURI with valid authority token.
2. Robot validates the authority token (RURI signed by RRF-registered authority key).
3. Robot notifies the owner via their configured channel (e.g. WhatsApp alert).
4. Robot packages the requested data (audit chain, transparency records, SBOM URL, firmware manifest URL).
5. Robot responds with `AUTHORITY_RESPONSE (42)` within the `expires_at` window.
6. If `expires_at` has passed, robot responds with `ERROR (8)` code `AUTHORITY_REQUEST_EXPIRED`.

!!! warning ""
    **Owner notification required:** Implementations MUST notify the robot owner whenever an `AUTHORITY_ACCESS` request is received, regardless of whether the request is valid. Silent authority access is a protocol violation.

---

## Rate Limits & Security

- Maximum 1 `AUTHORITY_ACCESS` request per authority per 24 hours (configurable).
- Authority tokens MUST be signed by a key registered in RRF under the `authority` namespace.
- Robots MUST NOT respond to requests from unregistered authorities — respond with `ERROR (8)` code `AUTHORITY_NOT_RECOGNIZED`.
- All authority interactions MUST be logged to the commitment chain (§16) with `TRANSPARENCY (16)` records.
- The owner is notified for every request, including rejected ones.

---

## Conformance Requirements

- **L5 (required):** Robot MUST have an `AUTHORITY_ACCESS (41)` handler registered.
- **L5 (required):** Handler MUST validate authority token before responding.
- **L5 (required):** Owner MUST be notified on every authority request.
- **L5 (required):** All authority interactions MUST be logged to the commitment chain.
- **All levels:** Unrecognized authority requests MUST be rejected with a logged error.

---

## ISO/TC 299 Normative References

RCAN v2.1 provisions are normatively aligned with the following ISO standards. Implementations claiming RCAN v2.1 conformance at L3+ satisfy the RCAN-level requirements; full ISO certification requires independent audit against the cited standard.

| Standard | Committee | RCAN Provision | Mapping |
|----------|-----------|----------------|---------|
| ISO 13482:2014 | ISO/TC 299 | §6 Safety invariants | RCAN ESTOP (type 7) → ISO 13482 protective stop. ESTOP MUST always be honored regardless of role level (§2). |
| ISO 10218-2:2011 | ISO/TC 299 | §2 RBAC | RCAN role levels (GUEST–M2M_TRUSTED) satisfy ISO 10218-2 access control requirements for collaborative robots. |
| ISO/IEC 42001:2023 | ISO/IEC JTC 1/SC 42 | §16 Transparency | RCAN audit chain + TRANSPARENCY records (type 16) satisfy ISO/IEC 42001 AI management system evidence requirements. |
| ISO 13482:2014 | ISO/TC 299 | §7 ConfidenceGate | RCAN confidence thresholds (ABORT_THRESHOLD, CONFIRM_THRESHOLD) align with ISO 13482 risk-based decision gates. |
| ISO/IEC 27001:2022 | ISO/IEC JTC 1/SC 27 | §9 Signing | RCAN Ed25519 signing satisfies ISO/IEC 27001 Annex A cryptographic controls. Signed RURI required at L2+ (§2.1). |
| ISO/IEC 27001:2022 | ISO/IEC JTC 1/SC 27 | §11 Firmware | Signed firmware manifests (§11) satisfy ISO/IEC 27001 software integrity controls. |

!!! info ""
    **ISO/TC 299 scope:** ISO/TC 299 (Robotics) is the primary standards body for robot safety. ISO 13482 covers personal care robots; ISO 10218-2 covers industrial robot integration. RCAN v2.1's ESTOP, confidence gate, and RBAC provisions are designed to be directly traceable to these requirements.

---

## DISCOVER `iso_conformance` Block

Robots MAY include an `iso_conformance` object in their `DISCOVER (9)` response payload to self-report which ISO standards they conform to. This block is informational only — it does not replace formal certification and MUST NOT be used as a substitute for independent audit.

```json
// DISCOVER (type 9) — optional iso_conformance block (v2.1)
// Present when the robot can demonstrate ISO standard compliance
{
  "type":    9,
  "payload": {
    "capabilities": ["status", "control", "navigate", "contribute"],
    "ruri":         "rcan://rrf.rcan.dev/opencastor/rpi5-hailo/bob-001?sig=...",
    "iso_conformance": {
      "iso_13482":  true,
      "iso_10218_2": false,
      "iso_42001":  true
    }
  }
}
```

### `iso_conformance` Fields

| Field | Type | Meaning |
|-------|------|---------|
| `iso_13482` | bool | ISO 13482:2014 — Personal care robot safety (ISO/TC 299). `true` if ESTOP, confidence gate, and risk assessment provisions are implemented. |
| `iso_10218_2` | bool | ISO 10218-2:2011 — Industrial robot safety and integration (ISO/TC 299). `true` for industrial/collaborative robot applications. |
| `iso_42001` | bool | ISO/IEC 42001:2023 — AI management system (ISO/IEC JTC 1/SC 42). `true` if audit chain, transparency records, and SBOM are implemented. |

Additional ISO standard fields MAY be added using reverse-domain notation keys (e.g. `iso.tc299.13482_ext: true`). Unknown keys MUST be ignored by receivers.
