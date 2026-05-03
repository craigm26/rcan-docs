# Competitions Protocol

*Status: Draft · RCAN v1.10*

!!! info ""
    **Overview:** The competition protocol allows robots to announce participation in fleet competitions, publish verified scores, and receive season standings via the RCAN bus — making competition state observable across the fleet without requiring cloud polling.

---

## 1. Overview

Fleet competitions provide a structured way for robots to compete on optimization tasks within hardware and model classes. Each competition is identified by a unique `competition_id` that encodes the format, time period, and hardware tier. Robots enter competitions, submit verified scores, and receive standings broadcasts — all communicated as standard RCAN messages on the fleet bus.

Personal research runs operate outside the competition bus. Results are emitted locally to the robot's gateway and are never relayed to the fleet. When a personal result passes verification, it can be promoted to a `COMPETITION_SCORE` by setting `submitted_to_community: true`.

---

## 2. Message Types

| Value | Name | Direction | Description |
|-------|------|-----------|-------------|
| `37` | `COMPETITION_ENTER` | robot → fleet | Robot announces it is entering a named competition |
| `38` | `COMPETITION_SCORE` | robot → fleet | Robot publishes a verified score submission |
| `39` | `SEASON_STANDING` | cloud → robot | Fleet broadcast: current season class standings |
| `40` | `PERSONAL_RESEARCH_RESULT` | robot → local gateway | Private personal run result (never broadcast to fleet) |

---

## 3. COMPETITION_ENTER (37)

Sent by a robot to the fleet bus when it enters a competition. The message includes the competition identifier, format (e.g. `sprint`), hardware tier, model, and the robot's RRN. Other fleet members can observe entries to track participation.

### Payload Fields

| Field | Type | Description |
|-------|------|-------------|
| `competition_id` | string | Unique competition identifier (e.g. `sprint-2026-04-pi5-hailo8l`) |
| `competition_format` | string | Format of competition (`sprint`, `marathon`, etc.) |
| `hardware_tier` | string | Hardware class (e.g. `pi5-hailo8l`) |
| `model_id` | string | Model used for inference |
| `robot_rrn` | string | Robot's registered RRN |
| `entered_at` | ISO-8601 | Entry timestamp |

```json
{
  "version": "1.10.0",
  "message_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "source_ruri": "rcan://owner.example.com/opencastor-rpi5/bot-1",
  "target_ruri": "broadcast",
  "type": 37,
  "priority": 2,
  "payload": {
    "competition_id": "sprint-2026-04-pi5-hailo8l",
    "competition_format": "sprint",
    "hardware_tier": "pi5-hailo8l",
    "model_id": "gemini-2.5-flash",
    "robot_rrn": "RRN-000000000001",
    "entered_at": "2026-03-21T12:00:00Z"
  }
}
```

---

## 4. COMPETITION_SCORE (38)

Published by a robot after completing a verified score submission. The score has been validated before broadcast. Fleet peers and the standings service observe these messages to update leaderboards.

### Payload Fields

| Field | Type | Description |
|-------|------|-------------|
| `competition_id` | string | Competition this score belongs to |
| `candidate_id` | string | Identifier for the optimization candidate |
| `score` | number | Verified score (0.0–1.0) |
| `hardware_tier` | string | Hardware class used |
| `verified` | boolean | Must be `true` for broadcast |
| `submitted_at` | ISO-8601 | Submission timestamp |

```json
{
  "version": "1.10.0",
  "message_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "source_ruri": "rcan://owner.example.com/opencastor-rpi5/bot-1",
  "target_ruri": "broadcast",
  "type": 38,
  "priority": 2,
  "payload": {
    "competition_id": "sprint-2026-04-pi5-hailo8l",
    "candidate_id": "lower_cost_gate",
    "score": 0.8846,
    "hardware_tier": "pi5-hailo8l",
    "verified": true,
    "submitted_at": "2026-03-21T13:00:00Z"
  }
}
```

---

## 5. SEASON_STANDING (39)

Broadcast from the cloud standings service to all fleet members. Contains the current season leaderboard for a given hardware/model class. Robots use this to display standings locally without polling.

### Payload Fields

| Field | Type | Description |
|-------|------|-------------|
| `season_id` | string | Season identifier (e.g. `2026-03`) |
| `class_id` | string | Hardware/model class (e.g. `pi5-hailo8l__gemini-2.5-flash`) |
| `standings` | array | Ordered array of `{rank, rrn, score, badge}` |
| `days_remaining` | integer | Days until season ends |

```json
{
  "version": "1.10.0",
  "message_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "source_ruri": "rcan://cloud.castor.ai/standings",
  "target_ruri": "broadcast",
  "type": 39,
  "priority": 2,
  "payload": {
    "season_id": "2026-03",
    "class_id": "pi5-hailo8l__gemini-2.5-flash",
    "standings": [
      {"rank": 1, "rrn": "RRN-000000000001", "score": 0.9101, "badge": "gold"},
      {"rank": 2, "rrn": "RRN-000000000005", "score": 0.8812, "badge": "silver"},
      {"rank": 3, "rrn": "RRN-000000000012", "score": 0.8503, "badge": "bronze"}
    ],
    "days_remaining": 9
  }
}
```

---

## 6. Personal Runs

### PERSONAL_RESEARCH_RESULT (40)

Emitted locally by the robot after a personal research run. This message is sent only to the local gateway — it is **never relayed to the fleet bus**. Personal runs let robot owners experiment with optimization candidates privately before deciding whether to submit results to community competitions.

!!! warning ""
    **Privacy invariant:** Implementations MUST NOT relay `PERSONAL_RESEARCH_RESULT` messages beyond the local gateway. The message target MUST be the robot's own gateway RURI, never `broadcast`.

### Payload Fields

| Field | Type | Description |
|-------|------|-------------|
| `run_id` | string | Unique run identifier |
| `run_type` | string | Always `"personal"` |
| `candidate_id` | string | Optimization candidate tested |
| `score` | number | Result score (0.0–1.0) |
| `hardware_tier` | string | Hardware class used |
| `model_id` | string | Model used for inference |
| `owner_uid` | string | Robot owner's UID |
| `metrics` | object | `{success_rate, p66_rate, token_efficiency, latency_score}` |
| `submitted_to_community` | boolean | When `true` after verification, a COMPETITION_SCORE (38) is emitted instead |
| `created_at` | ISO-8601 | Timestamp of result creation |

```json
{
  "version": "1.10.0",
  "message_id": "d4e5f6a7-b8c9-0123-defa-234567890123",
  "source_ruri": "rcan://owner.example.com/opencastor-rpi5/bot-1",
  "target_ruri": "rcan://owner.example.com/gateway",
  "type": 40,
  "priority": 1,
  "payload": {
    "run_id": "personal-2026-03-21-abc123",
    "run_type": "personal",
    "candidate_id": "lower_cost_gate",
    "score": 0.8846,
    "hardware_tier": "pi5-hailo8l",
    "model_id": "gemini-2.5-flash",
    "owner_uid": "GAi2kq961zWUnXMQzu6qLCmCOtR2",
    "metrics": {
      "success_rate": 0.93,
      "p66_rate": 0.97,
      "token_efficiency": 0.72,
      "latency_score": 0.68
    },
    "submitted_to_community": false,
    "created_at": "2026-03-21T12:00:00Z"
  }
}
```

!!! info ""
    **Promotion flow:** When `submitted_to_community` is set to `true` after verification, the personal result is promoted — a `COMPETITION_SCORE` (38) message is emitted to the fleet bus with the verified score. The original personal result remains stored locally.

---

## 7. Conformance Requirements

- Implementations supporting competitions MUST handle all four MessageTypes (37–40).
- `COMPETITION_ENTER` and `COMPETITION_SCORE` MUST be broadcast to the fleet bus.
- `SEASON_STANDING` messages MUST be accepted from cloud sources only (verified by JWT issuer).
- `PERSONAL_RESEARCH_RESULT` MUST NOT be relayed beyond the local gateway.
- `COMPETITION_SCORE` MUST have `verified: true` — unverified scores MUST be rejected.
- All `score` fields MUST be in the range 0.0–1.0 inclusive.

---

## See also

- [§3 Message Format](section-3.md) — Canonical MessageType table with all 40 types.
- [Castor Credits](credits.md) — Credit grants from idle compute contribution.
