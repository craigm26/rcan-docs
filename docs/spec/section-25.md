# §25 Post-Market Monitoring

*Version: v3.0*

Defines the `rcan-incidents-v1` schema — a persistent, append-only incident log for EU AI Act Art. 72 post-market monitoring obligations.

---

## Overview

EU AI Act Art. 72 requires providers of high-risk AI systems to actively monitor performance after deployment and report serious incidents to relevant national authorities. RCAN §25 defines a machine-readable incident log format (JSONL) and report schema that support this obligation.

!!! warning ""
    **Reporting deadlines:** Incidents with severity `life_health` must be reported within **15 days**. All other incidents within **90 days**. Both deadlines are surfaced in the generated report under `reporting_deadlines`.

---

## Incident Log Format (JSONL)

Incidents are stored as an append-only JSONL file at `~/.opencastor/incidents.jsonl` by default. Each line is a complete JSON incident entry.

```json
# Append-only JSONL — one incident per line
{"id":"550e8400-e29b-41d4-a716-446655440000","timestamp":"2026-04-11T10:30:00.000Z","severity":"life_health","category":"motor_stall","description":"Left drive motor stalled under load during navigation","system_state":{"estop_active":false,"driver":"pca9685","duty":0.85}}
{"id":"6ba7b810-9dad-11d1-80b4-00c04fd430c8","timestamp":"2026-04-11T14:15:00.000Z","severity":"other","category":"provider_timeout","description":"AI provider response exceeded 5s timeout during REASONING task","system_state":{"provider":"anthropic","task_category":"REASONING","timeout_ms":5000}}
```

### Entry Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | MUST | UUID v4 — unique identifier for deduplication. |
| `timestamp` | string | MUST | ISO-8601 UTC timestamp when the incident was recorded. |
| `severity` | string | MUST | `"life_health"` (15-day reporting deadline) or `"other"` (90-day deadline). |
| `category` | string | MUST | Short category tag (e.g. `"motor_stall"`, `"provider_timeout"`). |
| `description` | string | MUST | Human-readable description of what happened. |
| `system_state` | object | SHOULD | Snapshot of relevant system state at the time of incident. |

---

## Report Schema (rcan-incidents-v1)

```json
{
  "schema": "rcan-incidents-v1",
  "generated_at": "2026-04-11T15:00:00.000Z",
  "total_incidents": 2,
  "incidents_by_severity": { "life_health": 1, "other": 1 },
  "reporting_deadlines": {
    "life_health": "15 days from incident timestamp",
    "other": "90 days from incident timestamp"
  },
  "art72_note": "Providers must report serious incidents to the relevant national authority within the applicable deadline per EU AI Act Art. 72.",
  "incidents": [...]
}
```

---

## CLI Reference

```bash
# Record an incident
castor incidents record \
  --severity life_health \
  --category motor_stall \
  --description "Left drive motor stalled"

# List all incidents
castor incidents list

# Generate Art. 72 report
castor incidents report --output incidents-report-2026.json
```
