# §10 Autonomous Navigation

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** The `nav` capability exposes a simple dead-reckoning waypoint API. No SLAM or odometry hardware is required; timing is derived from the `physics` block in the RCAN config file.

---

## 10.1 Overview

RCAN navigation is intentionally simple: it is dead-reckoning only, meaning the robot uses pre-calibrated timing values to estimate how far it has moved. There is no map, no sensor fusion, and no odometry requirement.

This design choice makes the `nav` capability universally implementable on low-cost hardware. Robots with more capable localization systems MAY use their own algorithms internally while conforming to the RCAN waypoint API externally.

---

## 10.2 Physics Prerequisites

The `physics` block MUST include these fields for dead-reckoning navigation. Without them, a robot MUST NOT advertise the `nav` capability.

```yaml
physics:
  wheel_circumference_m:  0.21     # metres per full wheel rotation
  turn_time_per_deg_s:    0.008    # seconds to turn one degree at speed=1.0
```

- `wheel_circumference_m` — used to compute travel time for a given distance at a given speed.
- `turn_time_per_deg_s` — calibrated seconds to turn one degree at speed=1.0. Scales linearly with speed.

---

## 10.3 Waypoint API

```json
// POST /api/nav/waypoint
{
  "distance_m":  1.5,
  "heading_deg": 90,
  "speed":       0.6
}

// Response
{
  "ok":       true,
  "job_id":   "nav-a1b2c3",
  "eta_ms":   3200
}

// GET /api/nav/status
{
  "running":     true,
  "job_id":      "nav-a1b2c3",
  "distance_m":  1.5,
  "heading_deg": 90,
  "elapsed_ms":  1100
}
```

### Request fields

| Field | Type | Requirement | Description |
|-------|------|-------------|-------------|
| `distance_m` | number (float) | MUST | Distance to travel in metres. Negative = reverse. |
| `heading_deg` | number (float) | MUST | Absolute heading to face before driving (0 = current forward). |
| `speed` | number [0–1] | SHOULD | Normalised motor speed. Default: 0.5. |

---

## 10.4 Execution Model

1. Turn to `heading_deg` (relative to current facing) — blocks until complete
2. Drive forward `distance_m` at `speed` — blocks until complete
3. Explicitly stop motors (`driver.stop()`) in a `finally` block

The `finally` block in step 3 is a safety requirement: even if an exception is raised mid-navigation (including an e-stop signal), the motors MUST be halted before returning.

The response includes a `job_id` for polling via `GET /api/nav/status`. The status endpoint returns the running state, elapsed time, and the original target parameters.

---

## 10.5 Constraints & Safety

- **One job at a time:** Only one nav job may run at a time. A new `POST /api/nav/waypoint` while a job is running MUST return `409 Conflict`.
- **E-stop integration:** If an e-stop is triggered mid-nav, the `finally` block guarantees motor stop regardless of navigation state.
- **Depth integration:** Robots with the `vision` capability SHOULD check obstacle zones (§12) before and during navigation. A `nearest_cm` below the configured minimum MUST trigger an e-stop.
- **Speed bounds:** `speed` MUST be clamped to [0.0, 1.0]. Values outside this range MUST be rejected with a 400 response.
