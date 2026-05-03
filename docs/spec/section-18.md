# §18 Capability Advertisement Protocol

*Status: Draft · RCAN v1.3*

!!! info ""
    **Overview:** The Capability Advertisement Protocol defines how a robot declares its functional capabilities during connection and status reporting. It extends the legacy comma-separated `caps` string with a structured **Capability Object Map** that carries version, required flag, and typed parameters per capability.

---

## 18.1 Overview

The `caps` field in `CONNECT` and `STATUS` messages declares what a robot can do. Two formats are supported:

**Legacy** (still valid): Comma-separated capability names as a plain string: `"caps": "move,grip,speak"`. No version, no parameters, no required flag. Treated as version `"1.0"`, required `false`.

**Preferred** (Capability Object Map): Structured map with version, required flag, and typed params per capability: `"caps": {"move": {"version": "1.0", ...}}`. Enables negotiation, validation, and richer capability introspection.

Implementations MUST accept both formats. When producing capability advertisements, implementations SHOULD use the Capability Object Map format.

---

## 18.2 Capability Object Map Schema

The Capability Object Map is a JSON object where each key is a capability name and each value is a **Capability Descriptor**.

```json
{
  "caps": {
    "move": {
      "version": "1.0",
      "required": true,
      "params": {
        "max_velocity_m_s": 1.5,
        "max_angular_rad_s": 2.0,
        "kinematic_model": "differential"
      }
    },
    "grip": {
      "version": "1.0",
      "required": false,
      "params": {
        "max_force_n": 50.0,
        "grip_types": ["parallel", "vacuum"]
      }
    },
    "speak": {
      "version": "1.0",
      "required": false,
      "params": {
        "languages": ["en-US", "fr-FR"],
        "voices": ["nova", "alloy"]
      }
    },
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

### Capability Descriptor Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | string | SHOULD | Semantic version of this capability's interface (e.g. `"1.0"`, `"2.1"`). Defaults to `"1.0"` if omitted. |
| `required` | boolean | SHOULD | If `true`, the connecting client MUST support this capability to proceed. Defaults to `false`. |
| `params` | object | MAY | Capability-specific typed parameters (see §18.3). Omitted when no parameters are relevant. |

!!! info ""
    **Schema:** A JSON Schema for this structure is available at `/schemas/capability-object.json`.

---

## 18.3 Standard Capability Names

The following capability names are defined by the RCAN specification. Implementations MAY define additional capability names using reverse-DNS notation (e.g. `com.example.lidar`).

### `move` — Locomotion / navigation

| Param | Type | Description |
|-------|------|-------------|
| `max_velocity_m_s` | float | Maximum translational velocity in m/s |
| `max_angular_rad_s` | float | Maximum angular velocity in rad/s |
| `kinematic_model` | string enum | `"differential"` \| `"ackermann"` \| `"holonomic"` \| `"arm"` |

### `grip` — Gripper / end-effector

| Param | Type | Description |
|-------|------|-------------|
| `max_force_n` | float | Maximum gripping force in Newtons |
| `grip_types` | array of string | Supported grip modes (e.g. `"parallel"`, `"vacuum"`, `"magnetic"`) |

### `speak` — Text-to-speech / audio output

| Param | Type | Description |
|-------|------|-------------|
| `languages` | array of string | Supported BCP-47 language tags (e.g. `"en-US"`, `"fr-FR"`) |
| `voices` | array of string | Named voice identifiers available on this robot |

### `stream` — Sensor stream output

| Param | Type | Description |
|-------|------|-------------|
| `streams` | array of string | Available stream types: `"rgb"`, `"depth"`, `"imu"`, `"lidar"` |
| `max_fps` | int | Maximum frames per second across all active streams |

### `invoke` — Skill / behavior invocation (see [§19](section-19.md))

| Param | Type | Description |
|-------|------|-------------|
| `skills` | array of string | Names of invocable skills. See §19 for the INVOKE protocol. |

---

## 18.4 Usage in CONNECT / STATUS

The `caps` field appears in two message types:

- **CONNECT** — Robot advertises its capabilities at session start. The server MAY reject the connection (`4001`) if required capabilities are missing on the client side.
- **STATUS** — Robot MAY include `caps` in periodic STATUS messages to reflect capability changes (e.g. after hardware fault detection).

```json
{
  "type": "CONNECT",
  "ruri": "rcan://robot.local",
  "version": "1.2",
  "caps": {
    "move": {
      "version": "1.0",
      "required": true,
      "params": {
        "max_velocity_m_s": 1.5,
        "max_angular_rad_s": 2.0,
        "kinematic_model": "differential"
      }
    },
    "stream": {
      "version": "1.0",
      "required": false,
      "params": {
        "streams": ["rgb", "depth"],
        "max_fps": 30
      }
    }
  }
}
```

### Negotiation Rules

1. The server MUST NOT assume capabilities absent from the advertised map.
2. If a capability descriptor has `required: true` and the server cannot satisfy it, the server MUST respond with an error and SHOULD close the connection.
3. Clients MUST ignore unknown capability names in a received Capability Object Map (forward-compatibility).
4. Capability params are advisory — they describe what the robot can do, not what the client has requested.

---

## 18.5 Backwards Compatibility

The legacy comma-separated string format (`"caps": "move,grip,speak"`) remains fully valid and MUST be accepted by all implementations.

```json
// Legacy string form (still valid)
{"type": "CONNECT", "caps": "move,grip,speak"}

// New Capability Object Map (preferred)
{
  "type": "CONNECT",
  "caps": {
    "move": {"version": "1.0", "required": true, "params": {"max_velocity_m_s": 1.5}},
    "grip": {"version": "1.0", "required": false}
  }
}
```

### Legacy Interpretation Rules

When a legacy string is received, implementations MUST treat each named capability as if it had the following descriptor:

```
{ "version": "1.0", "required": false }  // no params
```

| Legacy String | Effective Capability Object Map |
|--------------|--------------------------------|
| `"move,grip"` | `{"move": {"version":"1.0","required":false}, "grip": {"version":"1.0","required":false}}` |
| `""` (empty) | `{}` (empty map — no capabilities) |

---

## 18.6 Capability Versioning

Capability versions follow **MAJOR.MINOR** semantic versioning (e.g. `"1.0"`, `"2.1"`).

- A **MINOR** bump indicates backwards-compatible additions (new optional params).
- A **MAJOR** bump indicates breaking changes (removed or renamed params, changed semantics).
- Clients that receive a capability with a higher MAJOR version than they support MUST treat it as an unknown capability.
- Custom capability names MUST use reverse-DNS format to avoid collisions with future RCAN standard names (e.g. `"com.example.custom_sensor"`).

!!! warning ""
    **Note:** The RCAN Working Group maintains the registry of standard capability names. Proposals for new standard capabilities should be submitted via the RCAN specification issue tracker.
