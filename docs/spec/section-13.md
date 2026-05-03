# §13 Telemetry Streaming

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** Implementations MUST expose a WebSocket endpoint at `/ws/telemetry` that pushes robot state at a configurable rate (default: 5 Hz). The telemetry stream aggregates loop metrics, battery state, obstacle data, provider status, and navigation state into a single JSON object per frame.

---

## 13.1 Overview

The telemetry stream is the primary real-time data channel between a robot and monitoring clients. It is push-based (server → client) and designed for dashboards, logging systems, and LLM context windows.

The default push rate is 5 Hz (200 ms interval). Implementations MAY support configurable rates via a query parameter, but MUST default to 5 Hz. Rates above 20 Hz are NOT RECOMMENDED as they may saturate LAN bandwidth for lightweight embedded clients.

---

## 13.2 WebSocket Endpoint

The endpoint is:

```
wss://robot:8000/ws/telemetry?token=<jwt>
```

- The port MUST match `rcan_protocol.port` from the robot's config (default: 8000).
- TLS (wss://) is RECOMMENDED for remote connections; ws:// is acceptable on LAN.
- The server MUST send JSON-encoded text frames. Binary frames are not defined for this endpoint.

---

## 13.3 Message Schema

```json
// WS /ws/telemetry?token=<jwt>
// Server pushes at 5 Hz (200 ms interval)
{
  "ts":              1735603215123,
  "loop_latency_ms": 412,
  "loop_count":      8423,
  "battery_v":       7.4,
  "provider":        "huggingface",
  "using_fallback":  false,
  "obstacles": {
    "left_cm":    45,
    "center_cm":  82,
    "right_cm":   38,
    "nearest_cm": 38
  },
  "nav_running":     false,
  "behavior_name":   null,
  "paused":          false
}
```

### Field requirements

| Field | Requirement | Type | Description |
|-------|-------------|------|-------------|
| `ts` | MUST | int (Unix ms) | Server timestamp at message emission. |
| `loop_latency_ms` | MUST | number | Duration of the last perception-action loop iteration. |
| `loop_count` | MUST | int | Total loop iterations since runtime start. |
| `paused` | MUST | bool | Whether the perception-action loop is currently paused. |
| `provider` | SHOULD | string | Active LLM provider name (e.g. `"huggingface"`). |
| `using_fallback` | SHOULD | bool | Whether the fallback provider is currently active (§14). |
| `battery_v` | SHOULD | number \| null | Battery voltage in Volts; null if not available. |
| `obstacles` | SHOULD | object \| null | Obstacle zone schema from §12. null if no depth hardware. |
| `nav_running` | SHOULD | bool | Whether a nav job is currently executing (§10). |
| `behavior_name` | SHOULD | string \| null | Name of the currently running behavior, or null (§11). |
| `contribute` | MAY | object \| null | Idle compute contribution status (v1.7). Fields: `enabled` (bool), `active` (bool), `coordinator` (string\|null), `project` (string\|null), `work_units_total` (int), `contribute_minutes_today` (int), `contribute_minutes_lifetime` (int). |

---

## 13.4 Authentication

The WebSocket endpoint SHOULD accept a `token` query parameter. The token MUST be a valid RCAN JWT (§5) with at least `status` scope.

- When `rcan_protocol.enable_jwt: true`, unauthenticated connections MUST be rejected with HTTP 401 during the WebSocket upgrade handshake.
- When `rcan_protocol.enable_jwt: false`, unauthenticated connections MAY be accepted (development/LAN mode only).
- Token expiry MUST be checked at connection time. Expired tokens MUST be rejected.
- Implementations SHOULD NOT re-validate the token on every push frame (once per connection is sufficient).

---

## 13.5 Cross-References

- **§12** — Obstacle zone schema (`obstacles` field)
- **§14** — Provider management (`provider`, `using_fallback` fields)
- **§10** — Navigation state (`nav_running` field)
- **§11** — Behavior state (`behavior_name` field)
- **§20** — Telemetry Field Registry (extended joint telemetry schema)
- **Appendix B** — WebSocket Transport Binding (TLS, framing, reconnect)
