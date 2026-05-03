# §27 Fundamental Rights Impact Assessment (FRIA) Protocol

*Version: v3.0*

Defines the `rcan-fria-v1` schema — a signed risk assessment document required for EU AI Act Art. 9(4) compliance for RCAN-registered robots operating at L2+ conformance or in human-proximate environments.

---

## Overview

EU AI Act Art. 9(4) requires providers of high-risk AI systems to conduct a Fundamental Rights Impact Assessment (FRIA) before deployment. RCAN §27 defines a machine-readable FRIA schema (`rcan-fria-v1`) that captures identified risks, applicable mitigations from the RCAN protocol, human oversight configuration, and a cryptographic signature binding the assessment to the robot's registered identity.

The signed FRIA document serves as the primary compliance artifact for EU AI Act registration (see [§26 EU Register Submission](section-26.md)), post-market monitoring ([§25](section-25.md)), and audit purposes. It links regulatory obligations to specific RCAN protocol mechanisms — HiTL gates (§16.3), confidence thresholds (§16.2), and authorization scopes (§8).

!!! info ""
    **Regulatory basis:** EU AI Act Art. 9(4) — Risk management system. FRIA documents are required for all Annex III high-risk AI system deployments and RCAN L2+ conformance deployments in human-proximate environments.

---

## Trigger Conditions

A FRIA document (`rcan-fria-v1`) **MUST** be generated and signed before deployment when any of the following conditions apply:

- **L2+ conformance deployments** — any robot operating at RCAN conformance level L2, L3, or L4.
- **EU AI Act Annex III deployments** — any system classified under Annex III of the EU AI Act (e.g. safety components of machinery, critical infrastructure, biometric identification).
- **Human-proximate environments** — robots operating within 2 metres of human bystanders as part of their normal operational envelope, regardless of conformance level.

!!! warning ""
    **Note:** A FRIA is strongly RECOMMENDED for all L1 deployments as best practice, even where not strictly required by regulation.

---

## Schema

```json
{
  "schema_version": "1.0",
  "rrn": "RRN-000000000001",
  "generated_at": "2026-04-10T00:00:00Z",
  "annex_iii_basis": "Category 3(a) — safety component of machinery",
  "conformance_level": "L2",
  "risk_entries": [
    {
      "id": "risk-001",
      "article": "Art. 9(4)",
      "description": "Foreseeable misuse: navigation in uncleared human-proximate zones",
      "severity": "high",
      "rcan_mitigation": "§16.3 HiTL gate on NAVIGATE actions; geofence enforcement",
      "residual_risk": "low"
    }
  ],
  "human_oversight_config": {
    "hitl_scopes": ["NAVIGATE", "MANIPULATE"],
    "confidence_thresholds": { "NAVIGATE": 0.85 }
  },
  "signed_by": "RRN-000000000001",
  "signature": "<ML-DSA-65 signature over canonical JSON>"
}
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schema_version` | string | MUST | Always `"1.0"` for `rcan-fria-v1` documents. |
| `rrn` | string | MUST | Robot Registration Number of the assessed system. |
| `generated_at` | string | MUST | ISO-8601 UTC timestamp of document generation. |
| `annex_iii_basis` | string | MUST | EU AI Act Annex III classification basis (e.g. `"Category 3(a) — safety component of machinery"`). |
| `conformance_level` | string | MUST | RCAN conformance level: `"L1"` \| `"L2"` \| `"L3"` \| `"L4"`. |
| `risk_entries` | array | MUST | One or more risk entries. MUST be non-empty. |
| `risk_entries[].id` | string | MUST | Unique risk identifier within this document (e.g. `"risk-001"`). |
| `risk_entries[].article` | string | MUST | Applicable EU AI Act article (e.g. `"Art. 9(4)"`). |
| `risk_entries[].description` | string | MUST | Human-readable description of the identified risk or foreseeable misuse. |
| `risk_entries[].severity` | string | MUST | `"low"` \| `"medium"` \| `"high"` \| `"critical"`. |
| `risk_entries[].rcan_mitigation` | string | MUST | RCAN protocol section(s) and mechanism(s) that mitigate this risk. |
| `risk_entries[].residual_risk` | string | MUST | `"low"` \| `"medium"` \| `"high"` — risk level after mitigation. |
| `human_oversight_config` | object | MUST | HiTL and confidence gate configuration for the deployed system. |
| `human_oversight_config.hitl_scopes` | array | MUST | Action scopes requiring human-in-the-loop approval (§16.3). |
| `human_oversight_config.confidence_thresholds` | object | SHOULD | Per-scope confidence thresholds below which HiTL is triggered (§16.2). |
| `signed_by` | string | MUST | RRN of the signing robot or operator identity. |
| `signature` | string | MUST | ML-DSA-65 signature over canonical JSON (sorted keys, no whitespace) of all fields except `"signature"`. |

---

## Signing

FRIA documents **MUST** be signed using the robot's ML-DSA-65 identity key (§9). The signature binds the assessment to the specific registered robot identity, ensuring the document cannot be reused for a different system or tampered with after signing.

### Signing Algorithm

- Algorithm: `ML-DSA-65` (CRYSTALS-Dilithium, NIST FIPS 204)
- Input: canonical JSON of the document with the `signature` field excluded
- Canonical JSON: keys sorted lexicographically, no whitespace between tokens
- The `signed_by` field **MUST** match the RRN in the document

```python
import json

def canonical_json(obj: dict) -> bytes:
    """Produce canonical JSON: sorted keys, no whitespace."""
    return json.dumps(obj, sort_keys=True, separators=(",", ":")).encode("utf-8")

# Exclude the 'signature' field before signing
def fria_signing_payload(fria: dict) -> bytes:
    payload = {k: v for k, v in fria.items() if k != "signature"}
    return canonical_json(payload)
```

!!! info ""
    **Key reuse:** The ML-DSA-65 key used for FRIA signing is the same identity key used for RCAN message signing (§9). No additional key material is required. The public key is resolvable via the RCAN Registry (§21) using the robot's RRN.

---

## Export Formats

FRIA documents are exported as UTF-8 encoded JSON files. Two standard filename conventions apply:

- **Unsigned:** `fria-{rrn}-{date}.json` — e.g. `fria-RRN-000000000001-20260410.json`
- **Signed:** `fria-{rrn}-{date}-signed.json` — e.g. `fria-RRN-000000000001-20260410-signed.json`

The signed FRIA file is the authoritative compliance artifact. When submitting to the EU AI database ([§26](section-26.md)), the signed FRIA **MUST** be referenced in the `fria_ref` field of the `rcan-eu-register-v1` submission package.

---

## CLI Reference

```bash
# Generate a FRIA document from a running OpenCastor config
castor fria generate \
  --config bob.rcan.yaml \
  --output fria-bob-2026.json

# Sign the generated FRIA with the robot's ML-DSA-65 identity key
castor fria sign \
  --input fria-bob-2026.json \
  --key ~/.opencastor/identity.key \
  --output fria-bob-signed.json

# Verify a signed FRIA document
castor fria verify --input fria-bob-signed.json
```

---

## Conformance Requirements

| Level | FRIA Requirement |
|-------|-----------------|
| L1 | RECOMMENDED. No mandatory FRIA, but generation tooling SHOULD be available. |
| L2 | MUST generate and sign a FRIA before deployment. FRIA MUST reference active HiTL scopes and confidence thresholds. |
| L3 | All L2 requirements plus FRIA MUST cover federated identity and cross-registry data flows as risk entries where applicable. |
| L4 | All L3 requirements plus signed FRIA MUST be registered in the RCAN Registry alongside the robot's RRN (§21). |

---

## Cross-References

- [§8 Authorization & Scopes](section-8.md) — authorization scopes referenced in risk entries
- [§16 AI Accountability](section-16.md) (§16.2 Confidence Gates) — per-scope confidence thresholds in `human_oversight_config`
- [§16 AI Accountability](section-16.md) (§16.3 HiTL Gate) — HiTL scopes listed in `human_oversight_config.hitl_scopes`
- [§21 Registry Integration](section-21.md) — RRN resolution and public key lookup for signature verification
- [§25 Post-Market Monitoring](section-25.md) — FRIA referenced in ongoing monitoring obligations
- [§26 EU Register Submission](section-26.md) — signed FRIA is the required input for EU AI database registration
