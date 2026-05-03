# §24 Instructions for Use

*Version: v3.0*

Defines the `rcan-ifu-v1` schema — a structured Instructions for Use document covering all eight fields required by EU AI Act Art. 13(3) for high-risk AI systems.

---

## Overview

EU AI Act Art. 13 requires providers of high-risk AI systems to accompany each system with clear and adequate instructions for use. Art. 13(3) specifies eight mandatory information categories. RCAN §24 defines a machine-readable IFU document that can be auto-generated from the RCAN config and deployment context, then provided to deployers alongside the FRIA.

The `annex_iii_basis` must be one of the ten valid Annex III categories (see [§22](section-22.md)). An invalid value causes generation to fail immediately.

---

## Art. 13(3) Field Coverage

| Field | Article | Content |
|-------|---------|---------|
| `provider_identity` | Art. 13(3)(a) | Provider identity: RRN, robot name, contact, RCAN version, AI model. |
| `intended_purpose` | Art. 13(3)(b) | Intended purpose: deployment description and Annex III basis. |
| `capabilities_and_limitations` | Art. 13(3)(c) | What the system can and cannot do; known limitations. |
| `accuracy_and_performance` | Art. 13(3)(d) | Performance evidence — links to §23 benchmark results. |
| `human_oversight_measures` | Art. 13(3)(e) | HiTL gates, ESTOP, confidence gates, operator override. |
| `known_risks_and_misuse` | Art. 13(3)(f) | Foreseeable misuse scenarios and mitigations. |
| `expected_lifetime` | Art. 13(3)(g) | Software support lifecycle and hardware dependency. |
| `maintenance_requirements` | Art. 13(3)(h) | Update cadence, conformance checks, incident logging. |

---

## Schema

```json
{
  "schema": "rcan-ifu-v1",
  "generated_at": "2026-04-11T09:00:00.000Z",
  "art13_coverage": [
    "provider_identity", "intended_purpose", "capabilities_and_limitations",
    "accuracy_and_performance", "human_oversight_measures",
    "known_risks_and_misuse", "expected_lifetime", "maintenance_requirements"
  ],
  "provider_identity": {
    "rrn": "RRN-000000000001",
    "rrn_uri": "rrn://org/robot/model/id",
    "robot_name": "my-robot",
    "provider_name": "Example Corp",
    "provider_contact": "safety@example.com",
    "rcan_version": "3.0",
    "agent_provider": "anthropic",
    "agent_model": "claude-sonnet-4-6"
  },
  "intended_purpose": {
    "description": "Indoor autonomous navigation for warehouse logistics",
    "annex_iii_basis": "safety_component",
    "deployment_context": "High-risk AI system under EU AI Act Annex III"
  },
  "capabilities_and_limitations": {
    "summary": "Universal robot runtime connecting LLM AI providers to physical hardware.",
    "known_limitations": [
      "AI provider responses subject to model confidence thresholds",
      "Physical limits enforced by BoundsChecker configuration",
      "HiTL authorization required for high-risk actions"
    ]
  },
  "accuracy_and_performance": {
    "note": "Quantified evidence via castor safety benchmark. Safety path P95: ESTOP 100ms, BoundsCheck 5ms, ConfidenceGate 2ms, FullPipeline 50ms."
  },
  "human_oversight_measures": {
    "hitl_gates": "HiTL authorization gates (§8) prevent autonomous high-risk actions",
    "estop": "Emergency stop halts all motion; P95 latency ≤ 100ms",
    "confidence_gates": "AI commands below confidence threshold are blocked",
    "override": "Operators can halt the system at any time via ESTOP"
  },
  "known_risks_and_misuse": {
    "foreseeable_misuse": [
      "Deployment outside declared intended use",
      "Operating beyond configured physical bounds",
      "Disabling HiTL gates without risk assessment"
    ],
    "mitigations": [
      "BoundsChecker enforces hard physical limits",
      "Conformance checks detect disabled safety features"
    ]
  },
  "expected_lifetime": {
    "software_support": "Subject to OpenCastor release lifecycle (YYYY.MM.DD versioning)",
    "hardware_dependent": true,
    "note": "Deployer is responsible for post-market monitoring per Art. 72 and §25"
  },
  "maintenance_requirements": {
    "software_updates": "Run castor upgrade for runtime updates",
    "conformance_checks": "Run castor validate before each deployment",
    "incident_logging": "Run castor incidents record for safety-relevant events"
  }
}
```

---

## CLI Reference

```bash
castor ifu generate \
  --config bob.rcan.yaml \
  --annex-iii-basis safety_component \
  --intended-use "Indoor autonomous navigation" \
  --output ifu-bob-2026.json
```
