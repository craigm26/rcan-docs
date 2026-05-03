# §16 AI Accountability

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** As RCAN robots become AI-driven rather than rule-driven, the reasoning layer becomes a first-class safety surface. §6 defines what was commanded; §16 defines requirements for recording *how* a command was produced, gating dispatch on model confidence, and requiring human authorization for high-risk actions.

---

## 16.1 Overview

All features in this section are backwards-compatible with v1.1 implementations. New audit fields are SHOULD; gate behavior is opt-in via config. Implementations that do not use AI inference are unaffected by this section.

The three mechanisms defined here are complementary:

- **Model Identity** — record which AI model produced a command and with what confidence.
- **Confidence Gates** — automatically block or escalate low-confidence commands.
- **HiTL Gates** — require explicit human approval for high-risk action types.

---

## 16.2 Model Identity in Audit Records

If a COMMAND message was produced by an AI inference call, implementations SHOULD include an `ai` block in the audit record alongside the existing §6 fields.

### Schema

| Field | Requirement | Type | Description |
|-------|-------------|------|-------------|
| `ai.provider` | MUST | string | Non-empty string identifying the LLM provider (e.g. `"anthropic"`, `"openai"`, `"ollama"`). |
| `ai.model` | MUST | string | Model identifier as returned by or configured for the provider. |
| `ai.model_version` | SHOULD | string | Model version or snapshot date (e.g. `"2026-02"`). |
| `ai.layer` | SHOULD | string | One of `reactive`, `fast`, or `planner`; or originating agent name for multi-agent systems. |
| `ai.confidence` | SHOULD | float [0–1] | Self-reported or harness-computed confidence. MUST be omitted (not defaulted to 1.0) if unavailable. |
| `ai.inference_latency_ms` | SHOULD | number | Wall-clock time for the inference call in milliseconds. |
| `ai.thought_id` | SHOULD | string | Unique ID enabling correlation with the full Thought log (§16.5). |
| `ai.escalated` | MUST if present | bool | `true` if produced after automatic escalation from a lower brain layer; `false` otherwise (do not omit). |

### Example extended audit record

```json
{
  "principal": "operator@rcan://local.rcan/opencastor/rover/abc123",
  "ruri": "rcan://local.rcan/opencastor/rover/abc123",
  "timestamp_ms": 1740721482934,
  "message_id": "a3f7c1d9",
  "event": "COMMAND",
  "outcome": "ok",
  "ai": {
    "provider": "anthropic",
    "model": "claude-sonnet-4-6",
    "model_version": "2026-02",
    "layer": "planner",
    "confidence": 0.87,
    "inference_latency_ms": 1240,
    "thought_id": "th_8f3a2c...",
    "escalated": false
  }
}
```

---

## 16.3 Confidence Gates

Implementations MAY declare confidence gates under `agent.confidence_gates` in the `.rcan.yaml` config file. A gate is a protocol-level constraint that blocks or escalates low-confidence commands before they reach the driver layer.

### Gate evaluation

1. Before dispatching any COMMAND with a gated scope, the runtime MUST evaluate the corresponding gate.
2. If `ai.confidence` is present and below `min_confidence`, the gate fails.
3. If `ai.confidence` is absent and a gate is declared for that scope, the gate defaults to the `on_fail` behavior (treat missing confidence as a gate miss).

### `on_fail` behaviors

| Value | Behavior |
|-------|----------|
| `block` | Command MUST NOT be dispatched. An ERROR response with code `CONFIDENCE_GATE_FAIL` MUST be returned. |
| `escalate` | Command MUST be re-submitted to the next higher brain layer before dispatch. If no higher layer is available, equivalent to `block`. |
| `allow` | Command proceeds. The gate miss MUST be noted in the audit record with `"gate_bypassed": true`. |

---

## 16.4 Human-in-the-Loop Gates

Implementations MAY declare Human-in-the-Loop (HiTL) gates under `agent.hitl_gates` in the `.rcan.yaml` config file. A HiTL gate requires explicit human authorization before a matched command is dispatched.

### Gate trigger sequence

When `require_auth: true` is declared and an incoming command's action type matches an entry in `action_types`:

1. The implementation MUST NOT dispatch the command immediately.
2. The implementation MUST emit a `PENDING_AUTH` message to all channels listed in `notify`, including the original `message_id`, `action_type`, a human-readable description, and the `auth_timeout_ms`.
3. The implementation MUST await an `AUTHORIZE` message from a principal with OWNER (level 4) or CREATOR (level 5) role within `auth_timeout_ms` milliseconds.
4. On `decision: approve` — the command is dispatched and both the original command and the `AUTHORIZE` message are logged in the audit chain.
5. On `decision: deny` or timeout expiry — the command is dropped, an `ERROR` is returned to the originating principal, and the event is audited.

### `on_timeout` behaviors

| Value | Behavior |
|-------|----------|
| `block` | Command is dropped; ERROR returned; event audited. |
| `allow` | Command is dispatched after timeout with `"hitl_timeout_bypass": true` in the audit record. |

!!! warning ""
    **Role enforcement:** `AUTHORIZE` messages from principals below OWNER role MUST be rejected. A rejected `AUTHORIZE` attempt MUST be audited (see §2). All gate events (trigger, approve, deny, timeout) MUST be written to the audit log with the original `message_id`.

---

## 16.5 Thought Log

Implementations SHOULD expose a queryable Thought log to enable full reasoning auditability and correlation with command audit records.

```json
GET /api/thoughts/<thought_id>
→ {
    "id": "th_8f3a2c...",
    "timestamp_ms": 1740721482100,
    "provider": "anthropic",
    "model": "claude-sonnet-4-6",
    "layer": "planner",
    "instruction": "Move toward the open doorway",
    "context_snapshot": { "...": "..." },
    "action": { "type": "move", "linear_x": 0.4 },
    "confidence": 0.87,
    "reasoning": "<omitted — requires config scope>",
    "escalated_from": null
  }
```

### Access control

- The endpoint MUST require at minimum `status` scope to read any Thought record.
- The `reasoning` field MUST be omitted from the response unless the caller holds `config` scope (OWNER or higher). This prevents prompt extraction by lower-privilege principals who have operational access but not configuration access.
- Implementations that do not expose the Thought log MUST return `404` or `501 Not Implemented` at this path.

---

## 16.6 AI Output Watermarking

*Version: v1.7 · Status: Stable*

EU AI Act Art. 50 requires that AI-generated content be machine-detectable. §16.6 specifies a cryptographic watermark token embedded in every AI-generated `COMMAND` payload and its corresponding audit record.

### Token Format

Watermark tokens use the prefix `rcan-wm-v1:` followed by 32 lowercase hex characters (16 bytes of HMAC-SHA256 output). The format is machine-detectable by regex: `^rcan-wm-v1:[0-9a-f]{32}$`

```python
import hashlib, hmac

def compute_watermark_token(rrn, thought_id, timestamp_iso, ml_dsa_private_bytes):
    message = f"{rrn}:{thought_id}:{timestamp_iso}".encode()
    digest = hmac.new(ml_dsa_private_bytes, message, hashlib.sha256).digest()
    return f"rcan-wm-v1:{digest[:16].hex()}"

# Example output:
# "rcan-wm-v1:a3f9c1d2b8e47f20a3f9c1d2b8e47f20"
```

### HMAC Key

The HMAC secret is the robot's ML-DSA-65 private key bytes (§1, 4032 bytes). This key is already present at runtime for message signing — no additional key material is required. The token proves the command originated from a robot with a specific identity; verification requires the robot's public verification endpoint (see below).

### Required Fields

Implementations at conformance level L2+ MUST include `watermark_token` in:

- The `COMMAND` message payload (§3)
- The corresponding audit record (§16.2)

```json
{
  "type": "COMMAND",
  "source": "rcan://robot.local:8000/bob",
  "payload": {
    "action": "move",
    "linear": 0.3,
    "angular": 0.0,
    "watermark_token": "rcan-wm-v1:a3f9c1d2b8e47f20a3f9c1d2b8e47f20"
  },
  "sig": { "alg": "ml-dsa-65", "kid": "a3f9c1d2", "value": "..." }
}
```

### Verification Endpoint

Implementations MUST expose a public (no authentication required) verification endpoint. The endpoint looks up the token in the tamper-evident audit log and returns the full audit entry, proving both token validity and that the command was logged.

```yaml
# Request
GET /api/v1/watermark/verify?token=rcan-wm-v1:a3f9c1d2b8e47f20a3f9c1d2b8e47f20&rrn=RRN-000000000001

# Response 200 — token found in audit log
{
  "valid": true,
  "rrn": "RRN-000000000001",
  "watermark_token": "rcan-wm-v1:a3f9c1d2b8e47f20a3f9c1d2b8e47f20",
  "audit_entry": {
    "ts": "2026-04-10T14:32:01.123456",
    "event": "motor_command",
    "source": "brain",
    "watermark_token": "rcan-wm-v1:a3f9c1d2b8e47f20a3f9c1d2b8e47f20",
    "ai": { "thought_id": "thought-abc123", "confidence": 0.91, "model": "claude-sonnet-4-6" },
    "action": { "type": "move", "linear": 0.3, "angular": 0.0 },
    "sig": { "alg": "ml-dsa-65", "kid": "a3f9c1d2", "value": "..." }
  }
}

# Response 404 — token not in audit log
{ "error": "Watermark token not found in audit log", "code": "HTTP_404" }

# Response 400 — malformed token format
{ "error": "Invalid watermark token format", "code": "HTTP_400" }
```

### EU AI Act Art. 50 Compliance

RCAN §16.6 satisfies the machine-detectability requirement for AI-generated content in robot command pipelines (EU AI Act Art. 50(2)). The `rcan-wm-v1:` prefix is detectable by regex without cryptographic operations. Full verification via the public endpoint proves both origin and audit chain membership, satisfying Art. 12 record-keeping requirements simultaneously.

### Conformance

| Level | Requirement |
|-------|-------------|
| L1 Core | Not required |
| L2 Secure | MUST embed token in COMMAND payload and audit record; MUST expose verify endpoint |
| L3 Federated | L2 requirements + token preserved in forwarded COMMAND across delegation chain |
| L4 Registry | L3 requirements + RRF registry MAY cache verify results for cross-robot auditability |
