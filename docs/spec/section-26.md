# §26 EU Register Submission

*Version: v3.0*

Defines the `rcan-eu-register-v1` schema — a submission package for the EU AI Act Art. 49 obligation to register high-risk AI systems in the EU AI database before placing them on the market.

---

## Overview

EU AI Act Art. 49 requires providers of high-risk AI systems to register their systems in the EU database administered by the European AI Office before placing the system on the market or putting it into service. RCAN §26 defines a submission package that wraps a signed FRIA document with the registration metadata required by the database.

Art. 49 registration is scoped **per AI system (per model)**, not per individual deployed robot — a provider with a fleet of the same model registers the model once. The top-level `rmn` field identifies which model this submission is for; `system.rrn` records which specific robot produced the submission (for audit/provenance only).

!!! warning ""
    **Prerequisite:** A valid signed `rcan-fria-v1` document is required as input. The submission package references it by filename and the two files must be submitted together to the EU AI database.

---

## Schema

```json
{
  "schema": "rcan-eu-register-v1",
  "generated_at": "2026-04-24T09:00:00.000Z",
  "rmn": "RMN-000000000007",
  "fria_ref": "fria-RRN-000000000001-20260411-signed.json",
  "provider": {
    "name": "Example Corp",
    "contact": "safety@example.com"
  },
  "system": {
    "rrn": "RRN-000000000001",
    "rrn_uri": "rrn://org/robot/model/id",
    "robot_name": "my-robot",
    "rcan_version": "3.0",
    "opencastor_version": "2026.4.24.0"
  },
  "annex_iii_basis": "safety_component",
  "conformity_status": "declared",
  "submission_instructions": "Submit this package to the EU AI Act database at https://ec.europa.eu/digital-strategy/en/policies/european-ai-act. Include the referenced FRIA JSON as an attachment."
}
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schema` | string | MUST | Always `"rcan-eu-register-v1"`. |
| `generated_at` | string | MUST | ISO-8601 UTC timestamp. |
| `rmn` | string | MUST | Robot Model Number — the model being registered under Art. 49. Format: `RMN-XXXXXXXXXXXX`. |
| `fria_ref` | string | MUST | Filename of the signed `rcan-fria-v1` document included with this submission. |
| `provider.name` | string | MUST | Provider organisation name. |
| `provider.contact` | string | MUST | Provider contact email for notified body queries. |
| `system.rrn` | string | MUST | Robot Registration Number of the specific robot that produced this submission (provenance). |
| `system.rcan_version` | string | MUST | RCAN protocol version implemented. |
| `annex_iii_basis` | string | MUST | Annex III classification basis (see [§22](section-22.md) for valid values). |
| `conformity_status` | string | MUST | `"declared"` — self-declared conformity assessment per Art. 43. |
| `submission_instructions` | string | MUST | Human-readable instructions for completing the national database submission. |

---

## CLI Reference

```bash
# Generate EU register submission package
# Requires a valid signed FRIA document as input
castor eu-register \
  --fria fria-bob-signed.json \
  --config bob.rcan.yaml \
  --output eu-register-bob-2026.json
```
