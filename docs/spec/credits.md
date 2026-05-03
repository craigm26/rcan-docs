# Castor Credits

*Idle Compute Contribution Protocol · Status: Draft · RCAN v1.7*

---

## 1. Overview

Castor Credits are earned by robots contributing idle compute capacity to harness research tasks — such as neural architecture search and model evaluation. The protocol is defined as part of RCAN v1.7 under the `contribute` scope (level 2.5).

Key properties:

- **1 credit ≈ 1 evaluated candidate.** Each accepted `CONTRIBUTE_RESULT` submission earns the robot's owner exactly one credit (subject to validation).
- **Tracked per owner, not per device.** Credits accumulate on the `owner_uid`, so a fleet of robots contributes to a shared pool.
- **Transferable within a fleet.** Credits may be moved between robots registered under the same `owner_uid`.
- **Non-transferable across owners.** Inter-owner credit transfer is not supported in v1.7.

!!! info ""
    **Scope requirement:** All three contribution message types require the `contribute` scope (level 2.5). Robots without this scope MUST reject contribution requests with an `ERROR` response (code: `SCOPE_INSUFFICIENT`).

---

## 2. Message Types

The contribute protocol uses three dedicated [§3](section-3.md) MessageTypes. All three require the `contribute` scope.

| Value | Name | Direction | Description |
|-------|------|-----------|-------------|
| `33` | `CONTRIBUTE_REQUEST` | Coordinator → Robot | Deliver a work unit for evaluation. Carries model format, input data, and timeout. |
| `34` | `CONTRIBUTE_RESULT` | Robot → Coordinator | Return evaluation results. Response payload includes a `credit_grant` object when the submission is accepted. |
| `35` | `CONTRIBUTE_CANCEL` | Robot → Coordinator | Cancel an active contribution (preempted by higher-priority command, battery low, etc.). |

### CONTRIBUTE_REQUEST (33)

Sent by the coordinator to a robot that has advertised idle compute capacity. The robot MUST respond with either a `CONTRIBUTE_RESULT` or `CONTRIBUTE_CANCEL` before the `timeout_s` expires.

```json
{
  "version": "1.7.0",
  "message_id": "550e8400-e29b-41d4-a716-446655440000",
  "source_ruri": "rcan://coordinator.castor.ai/prod",
  "target_ruri": "rcan://owner.example.com/robot-arm-7",
  "type": 33,
  "priority": 1,
  "payload": {
    "work_unit_id": "wu_2026031_0042",
    "project": "castor-architecture-search",
    "model_format": "onnx",
    "input_data": "...",
    "timeout_s": 120
  }
}
```

### CONTRIBUTE_RESULT (34)

Sent by the robot upon completing the work unit. When the coordinator validates the result, a `credit_grant` object is embedded in the acknowledgement payload. If validation fails, no credit is issued and an `ERROR` is returned.

```json
{
  "version": "1.7.0",
  "message_id": "660f9511-f30c-52e5-b827-557766551111",
  "source_ruri": "rcan://owner.example.com/robot-arm-7",
  "target_ruri": "rcan://coordinator.castor.ai/prod",
  "type": 34,
  "priority": 1,
  "payload": {
    "work_unit_id": "wu_2026031_0042",
    "output": { "loss": 0.0421, "accuracy": 0.9734 },
    "latency_ms": 8204,
    "hw_profile": { "npu_tops": 26, "tier": "silver" },
    "status": "complete",
    "credit_grant": {
      "owner_uid": "uid_abc123",
      "robot_rrn": "RRN-A1B2C3D4E5F6G7H8",
      "credits_earned": 1,
      "candidate_id": "wu_2026031_0042",
      "hardware_tier": "silver",
      "score": 0.9734,
      "timestamp": "2026-03-20T14:30:00Z"
    }
  }
}
```

### CONTRIBUTE_CANCEL (35)

Sent by the robot to abort an active contribution. Common reasons include P66 preemption (see §5 below), low battery, thermal throttle, or operator override. The `partial_result` field is optional; if present, the coordinator MAY use it for scoring even without a full result. No credit is issued for a cancelled work unit.

---

## 3. Credit Grant Schema

The `credit_grant` object is attached to the coordinator's `CONTRIBUTE_RESULT` acknowledgement when a submission is accepted. Implementations MUST validate all fields before persisting credits.

```json
{
  "owner_uid": "string",
  "robot_rrn": "string (RRN-XXXXXXXXXXXXXXXXX)",
  "credits_earned": "integer",
  "candidate_id": "string",
  "hardware_tier": "string",
  "score": "number (0.0–1.0)",
  "timestamp": "ISO-8601"
}
```

| Field | Type | Notes |
|-------|------|-------|
| `owner_uid` | string | Identity of the robot's registered owner. Credits are applied to this account. |
| `robot_rrn` | string | Canonical RRN of the contributing robot (format: `RRN-XXXXXXXXXXXXXXXXX`). |
| `credits_earned` | integer | Number of credits granted. Always 1 in v1.7; reserved for future batch work units. |
| `candidate_id` | string | Opaque identifier for the evaluated model candidate. Matches `work_unit_id`. |
| `hardware_tier` | string | Hardware capability tier of the contributing robot at evaluation time. |
| `score` | number | Validation score for the submitted result (0.0–1.0). Scores below 0.5 may result in credit withholding. |
| `timestamp` | string | ISO-8601 UTC timestamp when the credit was granted by the coordinator. |

---

## 4. Badge Tiers

Owners accumulate credits across their entire fleet. Badge tier is determined by the total lifetime credits earned by the `owner_uid`. Badges are displayed in the leaderboard and may be embedded in conformance reports.

| Badge | Credits | Description |
|-------|---------|-------------|
| Bronze | 100–499 | Entry contributor |
| Silver | 500–1,999 | Active contributor |
| Gold | 2,000–9,999 | Major contributor |
| Diamond | 10,000+ | Elite contributor |

!!! info ""
    **Tier downgrade:** Tiers are based on lifetime credits and never decrease. An owner who reaches Silver cannot fall back to Bronze regardless of future inactivity.

---

## 5. P66 Preemption

Contribution work runs at the lowest priority level (P1 — `LOW`). Any active robot command arriving at **priority P66 or higher** (i.e. `NORMAL`, `HIGH`, or `SAFETY`) immediately preempts an active contribution session.

The robot MUST send a `CONTRIBUTE_CANCEL` message to the coordinator within 500 ms of receiving the preempting command. The cancel reason SHOULD be set to `"preempted"`.

!!! warning ""
    **Safety invariant:** Contribution MUST NOT delay, buffer, or rate-limit any command at priority `NORMAL` or above. The `contribute` scope (level 2.5) is explicitly subordinate to all operational scopes. A robot MUST NOT enter a contribution session while a `SAFETY`-priority message is pending.

!!! danger ""
    **Non-compliance:** Failing to cancel within 500 ms of preemption is a conformance violation. Coordinators MUST treat late cancellations as an implicit cancel and MUST NOT issue credit for the work unit.

---

## 6. References

- [§3 Message Format](section-3.md) — Full MessageType table including CONTRIBUTE_REQUEST (33), CONTRIBUTE_RESULT (34), CONTRIBUTE_CANCEL (35)
- [Competitions Protocol](competitions.md) — Fleet competitions and standings
