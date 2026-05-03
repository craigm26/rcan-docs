# §12 Depth & Sensing

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** Robots with stereo depth cameras (e.g. Intel OAK-D, RealSense) SHOULD implement the depth sensing API. Implementations without depth hardware MUST return `{"available": false}`. The API provides obstacle zone readings and a colorized depth overlay image for visualization.

---

## 12.1 Overview

Depth sensing provides the robot's perception pipeline with structured obstacle awareness. Rather than exposing raw depth data (which would require large bandwidth), RCAN abstracts the depth frame into three horizontal zones: left, center, and right. Each zone reports the minimum depth (nearest obstacle) within that column range.

This abstraction is intentional: it is sufficient for basic collision avoidance and LLM-assisted navigation without requiring the controller to process full depth frames.

---

## 12.2 Obstacle Zone API

```json
// GET /api/depth/obstacles
{
  "available":    true,
  "left_cm":      45,
  "center_cm":    82,
  "right_cm":     38,
  "nearest_cm":   38,
  "timestamp_ms": 1735603215123
}

// GET /api/depth/frame
// Returns: JPEG image with JET colormap depth overlay (45% opacity)
// Content-Type: image/jpeg
```

### Fields

| Field | Type | Requirement | Description |
|-------|------|-------------|-------------|
| `available` | bool | MUST | `false` if no depth hardware is available. |
| `left_cm` | number (int) | SHOULD | Minimum depth in columns 0–W/3 (left third). |
| `center_cm` | number (int) | SHOULD | Minimum depth in columns W/3–2W/3 (centre third). |
| `right_cm` | number (int) | SHOULD | Minimum depth in columns 2W/3–W (right third). |
| `nearest_cm` | number (int) | SHOULD | Minimum across all three zones. |
| `timestamp_ms` | int (Unix ms) | MUST | Timestamp of this depth reading. |

---

## 12.3 Zone Definition

The depth frame is divided into three equal vertical thirds (by column count, not pixel count):

| Zone | Column Range | Field |
|------|-------------|-------|
| Left | Columns 0 – W/3 | `left_cm` |
| Center | Columns W/3 – 2W/3 | `center_cm` |
| Right | Columns 2W/3 – W | `right_cm` |

`nearest_cm` is the minimum value across all three zones. Implementations MUST report the minimum observed depth within each zone (not average or median).

---

## 12.4 Depth Overlay Image

The `GET /api/depth/frame` endpoint returns a JPEG image with the depth data overlaid on the RGB camera frame using the **JET colormap**:

- **Blue** — far objects
- **Green / Yellow** — mid-range objects
- **Red** — near objects (closest to the camera)

The overlay MUST be blended at approximately 45% opacity over the RGB frame so the scene remains recognizable. `Content-Type` MUST be `image/jpeg`.

---

## 12.5 Safety Integration

Implementations SHOULD integrate obstacle zone readings into the COMMAND pipeline:

- A `nearest_cm` below the configured `agent.min_obstacle_m × 100` value MUST trigger an e-stop before the motor command is dispatched.
- The e-stop triggered by obstacle detection MUST be logged in the audit trail (§6 Invariant 3).
- Robots without depth hardware MUST set `available: false` and MUST NOT perform obstacle checks (they cannot; this is a graceful-degradation requirement, not a safety bypass).

!!! warning ""
    **LLM integration:** The obstacle zones are included in the telemetry stream (§13) and can be passed to the LLM brain as context. The LLM SHOULD use zone readings to modulate navigation decisions, but protocol-level e-stops (above) are enforced independently of LLM output.
