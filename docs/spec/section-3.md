# §3 Message Format

*Status: Stable · RCAN v2.1*

All RCAN messages use a common envelope. Implementations MUST support JSON encoding; Protobuf encoding is RECOMMENDED for bandwidth-constrained links. The envelope carries identity, routing, auth, and type metadata — the payload is type-specific.

---

## 3.1 Overview

Every RCAN message — whether sent over WebSocket, HTTP, or a future transport — shares the same envelope structure. The envelope provides: sender and recipient identification (via RURI), authentication (JWT token), message type routing, a TTL for expiry, priority for queue ordering, and an optional reply-to address.

---

## 3.2 Message Envelope

The normative definition is expressed in Protobuf 3. JSON serialization follows field names exactly.

```protobuf
message RCANMessage {
  string version        = 1;   // "1.3.0"
  string message_id     = 2;   // UUID v4
  string source_ruri    = 3;   // Sender RURI
  string target_ruri    = 4;   // Recipient RURI or "broadcast"
  string auth_token     = 5;   // JWT session token (optional for DISCOVER)
  MessageType type      = 6;
  bytes payload         = 7;   // Type-specific JSON payload
  int64 timestamp_ms    = 8;
  int32 ttl_ms          = 9;   // Time-to-live (0 = no expiry)
  Priority priority     = 10;
  string reply_to       = 11;  // RURI to reply to (optional)
  repeated string scope = 12;  // Required scopes for this message

  // ── v2.1 required fields ─────────────────────────────────
  string firmware_hash   = 13;  // SHA-256 of sender's firmware manifest (REQUIRED v2.1)
  string attestation_ref = 14;  // URI to sender's /.well-known/rcan-sbom.json (REQUIRED v2.1)
  string delegation_chain = 15; // Serialized delegation chain (REQUIRED for COMMAND + INVOKE in v2.1)

  // ═══════════════════════════════════════════════════════════
  // v2.1 Canonical MessageType Table
  // This is the SINGLE SOURCE OF TRUTH. Both rcan-py and
  // rcan-ts MUST use these exact integer assignments.
  // ═══════════════════════════════════════════════════════════
  enum MessageType {
    // Core protocol (1–8)
    COMMAND            = 1;   // §3 — instruction to robot
    RESPONSE           = 2;   // §3 — response to command
    STATUS             = 3;   // §3 — robot state report
    HEARTBEAT          = 4;   // §3 — keepalive ping
    CONFIG             = 5;   // §8 — configuration update
    SAFETY             = 6;   // §6 — safety/ESTOP signal
    AUTH               = 7;   // §5 — authentication handshake
    ERROR              = 8;   // §3 — error response

    // Discovery & authorization (9–10)
    DISCOVER           = 9;   // §4 — mDNS/network discovery
    PENDING_AUTH       = 10;  // §5 — HiTL gate awaiting authorization (v1.2)

    // Skill invocation (11–13)
    INVOKE             = 11;  // §19.2 — skill/behavior invocation
    INVOKE_RESULT      = 12;  // §19.3 — result of INVOKE
    INVOKE_CANCEL      = 13;  // §19.4 — cancel an in-progress invocation

    // Registry (14–15)
    REGISTRY_REGISTER  = 14;  // §21.4 — register robot with RRF
    REGISTRY_RESOLVE   = 15;  // §21.5 — resolve RRN to RURI/metadata

    // Audit & transparency (16)
    TRANSPARENCY       = 16;  // §16 — EU AI Act Art. 13 audit record

    // Acknowledgement & QoS (17–18)
    COMMAND_ACK        = 17;  // QoS ≥ 1 acknowledgement
    COMMAND_NACK       = 18;  // Negative acknowledgement

    // Identity & consent (19–22)
    ROBOT_REVOCATION   = 19;  // Broadcast: revoke robot identity
    CONSENT_REQUEST    = 20;  // Request cross-robot consent
    CONSENT_GRANT      = 21;  // Owner grants consent
    CONSENT_DENY       = 22;  // Owner denies consent

    // Fleet & telemetry (23–25)
    FLEET_COMMAND      = 23;  // Broadcast command to robot group
    SUBSCRIBE          = 24;  // Subscribe to telemetry stream
    UNSUBSCRIBE        = 25;  // Cancel telemetry subscription

    // Diagnostics (26–28)
    FAULT_REPORT       = 26;  // Structured fault report
    KEY_ROTATION       = 27;  // Key rotation broadcast
    COMMAND_COMMIT     = 28;  // Exactly-once commit phase

    // Sensor & training data (29–32)
    SENSOR_DATA        = 29;  // Raw sensor payload
    TRAINING_CONSENT_REQUEST = 30;  // Training data consent request
    TRAINING_CONSENT_GRANT   = 31;  // Training data consent grant
    TRAINING_CONSENT_DENY    = 32;  // Training data consent deny

    // Idle compute contribution — v1.7 (33–35)
    CONTRIBUTE_REQUEST = 33;  // Coordinator → robot: deliver work unit
    CONTRIBUTE_RESULT  = 34;  // Robot → coordinator: return results
    CONTRIBUTE_CANCEL  = 35;  // Robot → coordinator: cancellation notice

    // Multimodal training data — v1.8 (36)
    TRAINING_DATA      = 36;  // Multi-modal training data payload

    // Competition — v1.10 (37–39)
    COMPETITION_ENTER  = 37;  // Robot announces competition entry
    COMPETITION_SCORE  = 38;  // Robot publishes verified score
    SEASON_STANDING    = 39;  // Cloud broadcasts season standings

    // Personal research — v1.10 (40)
    PERSONAL_RESEARCH_RESULT = 40;  // Local-only personal run result

    // Authority & attestation — v2.1 (41–44)
    AUTHORITY_ACCESS      = 41;  // Regulator → robot: request audit data (EU AI Act Art. 16(j))
    AUTHORITY_RESPONSE    = 42;  // Robot → authority: provide requested audit data
    FIRMWARE_ATTESTATION  = 43;  // Robot → RRF: publish signed firmware manifest
    SBOM_UPDATE           = 44;  // Robot → RRF: publish updated SBOM (CycloneDX)
  }

  enum Priority {
    LOW    = 1;
    NORMAL = 2;
    HIGH   = 3;
    SAFETY = 4;  // Always processed first; cannot be rate-limited
  }
}
```

---

## 3.3 Payload Types

The `type` field determines how the `payload` bytes are interpreted. Each type requires a minimum scope; messages without sufficient scope MUST be rejected with an `ERROR` response.

| MessageType | Required Scope | Payload Fields |
|------------|---------------|----------------|
| `COMMAND (1)` | control | instruction, image_b64 (optional) |
| `RESPONSE (2)` | — | ref_id, status, result |
| `STATUS (3)` | status | state, battery_v, loop_latency_ms |
| `HEARTBEAT (4)` | none | uptime_ms, sequence |
| `CONFIG (5)` | control | config_diff, scope, rollback_config — §8 |
| `SAFETY (6)` | safety | action ("estop"\|"resume"\|"fault"), reason — §6 |
| `AUTH (7)` | none | jwt_token, challenge, response — §5 |
| `ERROR (8)` | — | code, message, ref_id |
| `DISCOVER (9)` | none | capabilities[], ruri, iso_conformance{} (optional, v2.1) — §4 |
| `PENDING_AUTH (10)` | none | pending_id, action_type, description, timeout_remaining_ms |
| `INVOKE (11)` | control | skill_id (str), params (obj), timeout_ms (int) — §19.2 |
| `INVOKE_RESULT (12)` | — | reply_to, status, result (obj) — §19.3 |
| `INVOKE_CANCEL (13)` | control | payload.msg_id (str) — §19.4 |
| `REGISTRY_REGISTER (14)` | admin | rrn, ruri, metadata — §21.4 |
| `REGISTRY_RESOLVE (15)` | status | rrn — §21.5; response carries ruri, metadata |
| `TRANSPARENCY (16)` | status | model_id, decision, confidence, explanation — §16 |
| `COMMAND_ACK (17)` | — | ref_id, ok |
| `COMMAND_NACK (18)` | — | ref_id, reason, code |
| `ROBOT_REVOCATION (19)` | admin | rrn, reason, effective_at |
| `CONSENT_REQUEST (20)` | control | requested_scopes[], consent_type, requester_ruri |
| `CONSENT_GRANT (21)` | control | ref_id, granted_scopes[], expires_at |
| `CONSENT_DENY (22)` | control | ref_id, reason |
| `FLEET_COMMAND (23)` | control | group_id, instruction, target_filter |
| `SUBSCRIBE (24)` | status | stream_id, interval_ms, fields[] |
| `UNSUBSCRIBE (25)` | status | stream_id |
| `FAULT_REPORT (26)` | status | fault_code, severity, component, description |
| `KEY_ROTATION (27)` | admin | new_key_id, algorithm, effective_at |
| `COMMAND_COMMIT (28)` | — | ref_id, phase ("prepare"\|"commit"\|"rollback") |
| `SENSOR_DATA (29)` | status | sensor_type, readings[], timestamp_ms |
| `TRAINING_CONSENT_REQ (30)` | control | data_types[], purpose, retention_days |
| `TRAINING_CONSENT_GRANT (31)` | control | ref_id, approved_types[], conditions |
| `TRAINING_CONSENT_DENY (32)` | control | ref_id, reason |
| `CONTRIBUTE_REQUEST (33)` | contribute | work_unit_id, project, model_format, input_data, timeout_s |
| `CONTRIBUTE_RESULT (34)` | contribute | work_unit_id, output, latency_ms, hw_profile, status |
| `CONTRIBUTE_CANCEL (35)` | contribute | work_unit_id, reason, partial_result |
| `TRAINING_DATA (36)` | control | data_type, modality, payload_b64, consent_ref |
| `COMPETITION_ENTER (37)` | control | competition_id, competition_format, hardware_tier, model_id, robot_rrn, entered_at |
| `COMPETITION_SCORE (38)` | control | competition_id, candidate_id, score, hardware_tier, verified, submitted_at |
| `SEASON_STANDING (39)` | status | season_id, class_id, standings[], days_remaining |
| `PERSONAL_RESEARCH_RESULT (40)` | status | run_id, run_type, candidate_id, score, hardware_tier, model_id, owner_uid, metrics, submitted_to_community, created_at |
| `AUTHORITY_ACCESS (41)` | authority | request_id, authority_id, requested_data[], justification, expires_at |
| `AUTHORITY_RESPONSE (42)` | authority | request_id, rrn, data{}, provided_at |
| `FIRMWARE_ATTESTATION (43)` | admin | rrn, firmware_version, build_hash, components[], signed_at, signature |
| `SBOM_UPDATE (44)` | admin | rrn, cyclonedx_version, components[], x_rcan_firmware_hash, attestation_ref, signed_at |

---

## 3.4 Priority Levels

- **LOW (1)** — Background telemetry, non-urgent status updates.
- **NORMAL (2)** — Standard commands and queries.
- **HIGH (3)** — Time-sensitive commands; may preempt NORMAL queue.
- **SAFETY (4)** — Emergency stop, safety commands. MUST skip rate-limiting queues and be processed before any other message. Cannot be rate-limited per §6.

!!! warning ""
    **Safety invariant:** Messages with `Priority.SAFETY` bypass all rate limiting and queue ahead of all other messages. Implementations MUST NOT delay SAFETY messages regardless of queue backpressure.

---

## 3.5 Encoding Requirements

- Implementations MUST support JSON encoding for all message types.
- Protobuf 3 encoding is RECOMMENDED for bandwidth-constrained or high-frequency links.
- When using JSON, field names MUST match Protobuf field names exactly (no camelCase conversion).
- The `message_id` field MUST be a UUID v4. Implementations MUST reject messages with duplicate `message_id` values within a session (idempotency guard).
- `timestamp_ms` is Unix time in milliseconds. Implementations SHOULD validate that timestamps are within ±30 seconds of local time to guard against replay attacks.

---

## See also

- [Credits](credits.md) — Full protocol for CONTRIBUTE_REQUEST (33), CONTRIBUTE_RESULT (34), and CONTRIBUTE_CANCEL (35), including the credit grant schema and badge tiers.
- [Competitions](competitions.md) — Full protocol for COMPETITION_ENTER (37), COMPETITION_SCORE (38), SEASON_STANDING (39), and PERSONAL_RESEARCH_RESULT (40).
