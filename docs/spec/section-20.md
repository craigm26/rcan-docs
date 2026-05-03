# §20 Telemetry Field Registry

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** The Telemetry Field Registry defines canonical field names, types, and units for robot telemetry data carried in STATUS messages. Standardised field names enable interoperability across robot models, monitoring dashboards, and data pipelines without per-robot configuration.

---

## 20.1 Overview

Telemetry data in RCAN is embedded in STATUS messages. Two structured objects are defined:

- **Joint Telemetry Object** — Per-joint sensor readings for articulated robots (arms, legs, pan-tilt units).
- **Robot State Object** — Top-level robot state including power, position, and a joints array.

All telemetry fields use SI units unless otherwise specified. Field names use `snake_case` with unit suffixes (e.g. `position_rad`, `effort_nm`, `temperature_c`).

---

## 20.2 Joint Telemetry Object

A Joint Telemetry object represents a single snapshot of sensor readings for one named joint.

```json
{
  "joint": "shoulder_pan",
  "position_rad": 0.785,
  "velocity_rad_s": 0.02,
  "effort_nm": 0.41,
  "temperature_c": 38.5,
  "error_flags": 0,
  "timestamp_us": 1741737600000000
}
```

### Fields

| Field | Type | Required | Unit | Description |
|-------|------|----------|------|-------------|
| `joint` | string | MUST | — | Unique joint name within this robot (e.g. `"shoulder_pan"`, `"elbow_flex"`). MUST be stable across reboots. |
| `position_rad` | number (float) | MAY | rad | Current joint angle or linear position in radians. For prismatic joints, use `position_m` instead. |
| `velocity_rad_s` | number (float) | MAY | rad/s | Current joint velocity. Positive = increasing angle. |
| `effort_nm` | number (float) | MAY | N·m | Current joint torque or force. Positive = joint closing / forward effort. |
| `temperature_c` | number (float) | MAY | °C | Motor or actuator temperature in degrees Celsius. |
| `error_flags` | integer | MAY | bitmask | Driver-defined error bitmask. `0` = no errors. Bit definitions are hardware-specific; document in robot manifest. |
| `timestamp_us` | integer (int64) | MUST | µs | Hardware capture timestamp in microseconds since Unix epoch. MUST be monotonically increasing within a session. |

!!! info ""
    **Required fields:** Only `joint` and `timestamp_us` are required. All other fields are optional. Consumers MUST ignore unknown fields (forward-compatibility).

---

## 20.3 Robot State Object

The Robot State object is the top-level telemetry payload carried in a STATUS message. It aggregates robot-wide state and an optional array of Joint Telemetry objects.

```json
{
  "ruri": "rcan://robot.local",
  "seq": 1042,
  "uptime_ms": 3600000,
  "battery_v": 14.2,
  "battery_pct": 78,
  "position": {
    "x": 1.2,
    "y": 0.5,
    "heading_rad": 1.57
  },
  "joints": [
    {
      "joint": "shoulder_pan",
      "position_rad": 0.785,
      "velocity_rad_s": 0.02,
      "effort_nm": 0.41,
      "temperature_c": 38.5,
      "error_flags": 0,
      "timestamp_us": 1741737600000000
    }
  ],
  "timestamp_us": 1741737600000000
}
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ruri` | string (RURI) | MUST | Robot's canonical RURI. MUST match the RURI from CONNECT. |
| `seq` | integer | SHOULD | Monotonically increasing sequence number. Receivers MAY use for gap detection. |
| `uptime_ms` | integer | MAY | Milliseconds since robot process last started. |
| `battery_v` | number (float) | MAY | Battery pack voltage in Volts. |
| `battery_pct` | integer (0–100) | MAY | Battery state of charge as a percentage. |
| `position` | object | MAY | Robot's 2D world-frame position. Fields: `x` (float, m), `y` (float, m), `heading_rad` (float, rad). Origin is implementation-defined (typically map or odometry frame). |
| `joints` | array of Joint Telemetry | MAY | Array of Joint Telemetry objects (§20.2). MAY be empty or omitted if no joints are active. |
| `timestamp_us` | integer (int64) | MUST | Timestamp of this state snapshot in microseconds since Unix epoch. |

---

## 20.4 Extensibility Rules

The Telemetry Field Registry follows an open-world model:

1. **Consumers MUST ignore unknown fields.** New fields may be added in minor RCAN versions without a breaking change.
2. **Custom fields MUST use a vendor prefix.** Use reverse-DNS notation for custom fields to avoid collisions with future standard fields (e.g. `"com.example.motor_temp_c": 42.1`).
3. **Do not redefine standard field names.** A field named `position_rad` MUST always mean joint angle in radians. Repurposing standard names is a conformance violation.
4. **Unit suffixes are normative.** If you add a custom field with a unit, the suffix MUST accurately reflect the unit. Do not use `_rad` for a degree value.

!!! warning ""
    **Prismatic joints:** For linear joints (e.g. a lifting column), use `position_m` (metres) instead of `position_rad`, and `velocity_m_s` instead of `velocity_rad_s`.

---

## 20.5 Prometheus Label Convention

Implementations that export telemetry to Prometheus SHOULD use the following metric naming convention. The metric name prefix is **implementation-defined** — implementations SHOULD use a consistent prefix that identifies the exporter (e.g., the robot runtime name):

```
<prefix>_<subsystem>_<field>_<unit>{joint="<name>"}
```

Where:

- `<prefix>` — Implementation-defined prefix identifying the exporter (e.g. `rcan`, `myrobot`).
- `<subsystem>` — Robot subsystem identifier (e.g. `acb` for actuator control board).
- `<field>` — Field name without unit suffix (e.g. `position`, `velocity`, `effort`).
- `<unit>` — SI unit abbreviation (e.g. `rad`, `nm`, `c`).
- `joint="<name>"` — Label identifying the joint.

```text
# Joint telemetry — Prometheus exposition format
# NOTE: metric name prefix is implementation-defined (shown here as rcan_)
rcan_acb_position_rad{joint="shoulder_pan",ruri="rcan://robot.local"} 0.785
rcan_acb_velocity_rad_s{joint="shoulder_pan",ruri="rcan://robot.local"} 0.02
rcan_acb_effort_nm{joint="shoulder_pan",ruri="rcan://robot.local"} 0.41
rcan_acb_temperature_c{joint="shoulder_pan",ruri="rcan://robot.local"} 38.5

# Robot-level metrics
rcan_robot_battery_v{ruri="rcan://robot.local"} 14.2
rcan_robot_battery_pct{ruri="rcan://robot.local"} 78
rcan_robot_uptime_ms{ruri="rcan://robot.local"} 3600000
```

### Robot-Level Labels

Robot-level metrics (battery, uptime, position) use the `ruri` label instead of `joint`:

```
<prefix>_robot_battery_pct{ruri="rcan://robot.local"} 78
```

---

## 20.6 Timestamp Requirements

All telemetry timestamps use **microseconds since Unix epoch** (int64) stored in the `timestamp_us` field.

| Requirement | Level |
|-------------|-------|
| Hardware capture timestamp in `timestamp_us` | MUST |
| Monotonically increasing within a session | MUST |
| Synchronised to a time source (NTP, PTP, GPS) | SHOULD |
| Clock accuracy ≤ 1 ms for joint telemetry | SHOULD |
| Include `clock_source` field for diagnostics (`"ntp"`, `"ptp"`, `"rtc"`) | MAY |

!!! info ""
    **int64 range:** Microsecond-resolution int64 timestamps are valid until the year 294246. No Y2K-style overflow concern for practical robotics applications.
