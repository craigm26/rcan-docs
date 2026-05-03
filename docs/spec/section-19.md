# §19 Behavior/Skill Invocation Protocol

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** The Behavior/Skill Invocation Protocol defines how a client requests execution of a named skill on a robot, and how the robot reports the outcome. Skills are named, parameterised behaviors (e.g. `pick_and_place`, `patrol_loop`). The robot advertises available skills via the `invoke` capability in §18.

---

## 19.1 Overview

A **skill** is a named, self-contained behavior unit that a robot can execute. Skills accept typed parameters and produce a result. The Skill Invocation Protocol consists of three message types:

- **INVOKE** — Sent by the client to request execution of a named skill.
- **INVOKE_RESULT** — Sent by the robot when the skill completes, fails, or times out.
- **INVOKE_CANCEL** — Sent by the client to cancel an in-progress skill invocation (see §19.4).

Skills are synchronous from the protocol perspective: each INVOKE expects exactly one INVOKE_RESULT correlated via `msg_id` / `reply_to`. Long-running skills SHOULD emit intermediate STATUS messages but MUST produce a final INVOKE_RESULT.

---

## 19.2 INVOKE Message

The INVOKE message is sent by a client to request execution of a named skill. The robot MUST respond with an INVOKE_RESULT within `timeout_ms` milliseconds (or 30 000 ms if omitted).

```json
{
  "type": "INVOKE",
  "skill": "pick_and_place",
  "params": {
    "target": "red_cube"
  },
  "timeout_ms": 5000,
  "msg_id": "invoke_abc123"
}
```

### INVOKE Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | MUST | Literal `"INVOKE"` |
| `skill` | string | MUST | Name of the skill to invoke (e.g. `"pick_and_place"`). MUST match an entry in the `invoke.params.skills` capability array. |
| `params` | object | MAY | Skill-specific input parameters. Schema is defined per skill. Omit if the skill takes no params. |
| `timeout_ms` | integer | MAY | Maximum milliseconds to wait for INVOKE_RESULT. Defaults to `30000`. If exceeded, robot MUST send INVOKE_RESULT with status `"timeout"`. |
| `msg_id` | string | MUST | Unique message ID (UUIDv4 recommended). Used by INVOKE_RESULT to correlate the response via `reply_to`. |

!!! warning ""
    **Normative: msg_id Fallback Behavior**

    - Clients SHOULD always set `msg_id`; receivers MUST echo it in `INVOKE_RESULT`.
    - If a sender omits `msg_id` (non-conformant but tolerated), the receiver MUST generate a UUID v4 correlation ID internally and use it as the `reply_to` value in the paired `INVOKE_RESULT`.
    - `INVOKE_RESULT` MUST echo back the `msg_id` from the paired `INVOKE`, or the internally generated UUID if the client omitted one.
    - If no correlation is possible (e.g. fire-and-forget delivery with no session context), `INVOKE_RESULT` MAY omit `reply_to`. Implementations SHOULD log a warning in this case.

---

## 19.3 INVOKE_RESULT Message

The INVOKE_RESULT message is sent by the robot when a skill execution concludes (success, failure, or timeout). It MUST include a `reply_to` value equal to the `msg_id` from the originating INVOKE (or an internally generated UUID v4 if the client omitted `msg_id`).

```json
{
  "type": "INVOKE_RESULT",
  "skill": "pick_and_place",
  "status": "success",
  "reply_to": "invoke_abc123",
  "duration_ms": 1230
}
```

```json
{
  "type": "INVOKE_RESULT",
  "skill": "pick_and_place",
  "status": "failure",
  "reply_to": "invoke_abc123",
  "duration_ms": 312,
  "error": {
    "code": 7006,
    "name": "SkillFailed",
    "message": "Gripper could not secure the target object"
  }
}
```

```json
{
  "type": "INVOKE_RESULT",
  "skill": "undefined_skill",
  "status": "not_found",
  "reply_to": "invoke_xyz999",
  "error": {
    "code": 7001,
    "name": "SkillNotFound",
    "message": "No skill registered with name 'undefined_skill'"
  }
}
```

### INVOKE_RESULT Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | MUST | Literal `"INVOKE_RESULT"` |
| `skill` | string | MUST | Name of the skill that was invoked (mirrors the INVOKE `skill` field) |
| `status` | string (enum) | MUST | Outcome of the skill execution. See §19.5. |
| `reply_to` | string | MUST | The `msg_id` from the originating INVOKE, or the internally generated UUID v4 if the client omitted `msg_id`. MAY be omitted only when no correlation is possible (see §19.2 fallback note). |
| `duration_ms` | integer | MAY | Wall-clock time the skill ran in milliseconds, from start to completion. |
| `result` | object | MAY | Skill-specific output data. Schema defined per skill. Present on `"success"` when the skill produces output. |
| `error` | object | MAY | Error detail object with `code` (int), `name` (string), `message` (string). Present on non-success statuses. |

---

## 19.4 INVOKE_CANCEL Message

The INVOKE_CANCEL message allows a client to request cancellation of an in-progress skill invocation. It is an optional message type that enables cooperative abort for long-running skills.

```json
{
  "type": "INVOKE_CANCEL",
  "payload": {
    "msg_id": "abc-123",
    "reason": "User aborted navigation task",
    "cancel_timeout_ms": 5000
  }
}
```

### INVOKE_CANCEL Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | MUST | Literal `"INVOKE_CANCEL"` |
| `payload.msg_id` | string | MUST | The `msg_id` of the original INVOKE message to cancel. MUST match exactly. |
| `payload.reason` | string | OPTIONAL | Human-readable description of why the cancellation was requested. For logging and diagnostics only. |
| `payload.cancel_timeout_ms` | integer | OPTIONAL | Milliseconds the receiver will wait for graceful abort before force-terminating. Defaults to `5000`. |

### Normative Behavior

- Client sends INVOKE_CANCEL with `msg_id` matching the in-flight INVOKE.
- Receiver MUST attempt to abort the skill gracefully, allowing safe actuator wind-down within `cancel_timeout_ms` (default 5000 ms).
- Receiver MUST send INVOKE_RESULT with `status="cancelled"` after aborting (error code 7007).
- If the skill has already completed before INVOKE_CANCEL arrives, the receiver MUST silently ignore the INVOKE_CANCEL. The previously sent INVOKE_RESULT stands.
- If `msg_id` does not match any in-flight INVOKE, the receiver MUST respond with INVOKE_RESULT `status="not_found"` (error code 7001).
- INVOKE_CANCEL does not require a direct acknowledgement beyond the eventual INVOKE_RESULT.

```json
{
  "type": "INVOKE_RESULT",
  "skill": "navigate",
  "status": "cancelled",
  "reply_to": "abc-123",
  "duration_ms": 812,
  "error": {
    "code": 7007,
    "name": "SkillCancelled",
    "message": "Skill aborted by client INVOKE_CANCEL"
  }
}
```

### Conformance

| Message | L1 | L2 | L3 |
|---------|----|----|-----|
| `INVOKE_CANCEL` | MAY | SHOULD | MUST |

!!! warning ""
    **Implementation Note:** INVOKE_CANCEL is a best-effort cooperative mechanism. Skills that interact with physical actuators MUST ensure a safe state before emitting the cancelled INVOKE_RESULT, regardless of `cancel_timeout_ms`. Safety constraints (§16 HiTL gates, confidence gating) remain active during cancellation.

---

## 19.5 Status Values

The `status` field of INVOKE_RESULT MUST be one of the following enumerated values.

| Status | Meaning |
|--------|---------|
| `success` | The skill completed normally and achieved its goal. The `result` field MAY contain output data. |
| `failure` | The skill ran but did not achieve its goal (e.g. object not graspable). The `error` field SHOULD describe why. |
| `timeout` | The skill did not complete within `timeout_ms`. The robot aborted execution. Error code 7002. |
| `not_found` | No skill with the given name is registered on this robot. Error code 7001. |
| `unauthorized` | The session does not have sufficient scope or role to invoke this skill. Error code 7003. |
| `invalid_params` | The `params` object failed validation against the skill's parameter schema. Error code 7004. |
| `cancelled` | The skill was aborted in response to an INVOKE_CANCEL message. Error code 7007. See §19.4. |

---

## 19.6 Error Codes

The following error codes are reserved for the Skill Invocation Protocol. They supplement the general RCAN error codes.

| Code | Name | Status | Meaning |
|------|------|--------|---------|
| `7001` | `SkillNotFound` | `not_found` | No skill with the requested name is registered on this robot. The robot MUST NOT execute any action. |
| `7002` | `SkillTimeout` | `timeout` | Skill execution exceeded the requested `timeout_ms`. The robot MUST attempt a graceful abort of the skill. |
| `7003` | `SkillUnauthorized` | `unauthorized` | The session token lacks the scope or role required to invoke this skill. The robot MUST NOT execute any action. |
| `7004` | `InvalidSkillParams` | `invalid_params` | One or more params failed schema validation. The `error.message` field SHOULD identify the failing field. |
| `7005` | `SkillConflict` | `failure` | Another skill is currently running that conflicts with the requested skill. The client SHOULD retry after the conflicting skill completes. |
| `7006` | `SkillFailed` | `failure` | The skill ran but encountered a runtime error or could not achieve its goal. The `error.message` field SHOULD describe the cause. |
| `7007` | `SkillCancelled` | `cancelled` | The skill was aborted because the client sent INVOKE_CANCEL for the originating INVOKE. See §19.4. |

---

## 19.7 Integration with §18 Capability Advertisement

A robot that supports skill invocation MUST include the `invoke` capability in its Capability Object Map (§18). The `skills` parameter lists the names of all invocable skills.

```json
{
  "caps": {
    "invoke": {
      "version": "1.0",
      "required": false,
      "params": {
        "skills": ["pick_and_place", "patrol_loop", "door_open"]
      }
    }
  }
}
```

### Rules

1. A client MUST NOT send INVOKE for a skill not listed in `invoke.params.skills`. If it does, the robot MUST respond with status `not_found` (7001).
2. If the robot does not advertise the `invoke` capability, the client MUST NOT send INVOKE messages.
3. The skills list in `invoke.params.skills` is authoritative at the time of CONNECT. If skills change at runtime, the robot SHOULD emit a STATUS update with the revised capability map.
4. Skill names MUST be lowercase, using underscores as word separators (e.g. `pick_and_place`). Custom skills from third parties MUST use reverse-DNS prefix (e.g. `com.example.custom_skill`).

!!! info ""
    **Tip:** Clients should inspect the `invoke.params.skills` list after CONNECT_ACK and build their available-skills UI from it, rather than hard-coding skill names.

---

## 19.8 MessageType Reference

The following MessageType enum values are defined for the Skill Invocation Protocol. These values map to the `MessageType` enum in the §3 protobuf envelope.

| Value | Name | Direction | Description |
|-------|------|-----------|-------------|
| `11` | `INVOKE` | Client → Robot | Request execution of a named skill (§19.2) |
| `12` | `INVOKE_RESULT` | Robot → Client | Final outcome of a skill execution (§19.3) |
| `15` | `INVOKE_CANCEL` | Client → Robot | Cancel an in-progress skill invocation (§19.4) |
