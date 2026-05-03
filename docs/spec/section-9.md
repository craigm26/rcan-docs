# §9 Capabilities

*Status: Stable · RCAN v1.3*

Capabilities are advertised in `rcan_protocol.capabilities` and in mDNS TXT records. They control which RCAN message types are accepted and what scope is required. A robot MUST only accept messages for capabilities it has declared.

---

## 9.1 Overview

Capabilities are the primary mechanism for clients to discover what a robot can do without probing individual endpoints. They appear in three places: the RCAN config file (`rcan_protocol.capabilities`), the mDNS TXT record (`caps` field), and the DISCOVER message response.

A robot MUST NOT accept COMMAND messages for undeclared capabilities. For example, a robot without the `arm` capability MUST reject arm control commands with an `ERROR` response.

---

## 9.2 Standard Capabilities

| Capability | Scope Required | Description | Standard Endpoint |
|-----------|---------------|-------------|-----------------|
| `status` | STATUS | Health, telemetry, sensor readings | GET /api/status |
| `nav` | CONTROL | Autonomous waypoint navigation | POST /api/nav/waypoint |
| `teleop` | CONTROL | Direct real-time motor commands | POST /api/action |
| `vision` | STATUS | Camera stream, depth overlay | GET /api/stream/mjpeg |
| `chat` | CONTROL | NL command → brain → action | POST /api/command |
| `arm` | CONTROL | Manipulator/gripper control | POST /api/action |

---

## 9.3 Auto-Detection Rules

When `rcan_protocol.capabilities` is omitted from the config, the runtime SHOULD auto-detect capabilities from other config blocks:

| Config Condition | Auto-Declared Capabilities |
|-----------------|--------------------------|
| driver with `protocol: pca9685` or `protocol: dynamixel` | nav + teleop |
| `drivers` entry with `protocol: dynamixel` and kinematics | arm |
| `camera` block present | vision |
| `agent` block present | chat |
| Always | status |

---

## 9.4 Custom Capabilities

Implementations MAY declare custom capabilities using reverse-domain notation: `com.acme.gripper`, `io.opencastor.lidar`, etc.

- Custom capabilities MUST require at least `control` scope.
- Custom capability names MUST NOT collide with standard names.
- Custom capabilities SHOULD be documented in the robot's manifest or RCAN config comments.
- The same reverse-domain notation used for custom capabilities applies to custom mDNS TXT `caps` entries (§4).

---

## 9.5 Advertisement

Capabilities MUST be advertised consistently in all three locations:

1. **RCAN config** — `rcan_protocol.capabilities: [nav, vision, chat, teleop, status]`
2. **mDNS TXT** — `caps=nav,vision,chat,teleop,status` (§4)
3. **DISCOVER response** — `capabilities[]` field in the DISCOVER payload (§3)

Inconsistency between these three sources is a conformance violation. If the runtime detects a discrepancy at startup, it SHOULD log a warning and use the config file as authoritative.
