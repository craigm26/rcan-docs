# §11 Behavior Scripts

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** Behavior scripts are YAML files that define named sequences of robot steps. They provide repeatable, auditable automation without custom code — suitable for demos, patrols, inspection runs, and AI-assisted workflows.

---

## 11.1 Overview

A behavior script is a YAML file with a `name` and a `steps` array. Each step maps to a standard action type. The runtime executes steps sequentially; no branching or loops are defined in v1.3 (future versions may add conditional logic via the `think` step result).

Behaviors are auditable: each step execution is logged with a timestamp and outcome, providing a complete trace of automation activity. This makes behavior scripts suitable for environments that require operational audit trails (see §6 Invariant 3).

---

## 11.2 Script Format

```yaml
# behavior.yaml — RCAN Behavior Script v1.1
name: patrol_loop

steps:
  - type: speak
    text: "Starting patrol"

  - type: waypoint
    distance_m:  1.0
    heading_deg: 0
    speed:       0.5

  - type: think
    instruction: "Describe what you see"

  - type: wait
    seconds: 2

  - type: waypoint
    distance_m:  1.0
    heading_deg: 180   # turn around

  - type: speak
    text: "Patrol complete"

  - type: stop
```

The top-level `name` field MUST match the behavior name used in API calls. The `steps` array MUST contain at least one step. Files SHOULD use the extension `.behavior.yaml` for clarity.

---

## 11.3 Standard Step Types

| Type | Required Fields | Description |
|------|----------------|-------------|
| `waypoint` | `distance_m`, `heading_deg` | Dead-reckoning move (§10). `speed` is optional. |
| `wait` | `seconds` | Pause execution for the specified number of seconds. |
| `think` | `instruction` | Send NL prompt to brain; result available as `last_thought`. |
| `speak` | `text` | TTS output via audio system. |
| `stop` | — | Motor e-stop; also terminates behavior execution. |
| `command` | `action` (dict) | Send raw action dict directly to driver. |

!!! info ""
    **think step:** The result of a `think` step is stored as `last_thought` and MAY be referenced by subsequent steps. The instruction is passed verbatim to the configured LLM provider (§14). All prompt injection checks (§6 Invariant 7) apply.

---

## 11.4 Behavior API

```http
POST /api/behavior/run    { "path": "behaviors/patrol.yaml", "behavior": "patrol_loop" }
POST /api/behavior/stop
GET  /api/behavior/status → { "running": true, "current_step": 2, "behavior_name": "patrol_loop" }
```

The `run` endpoint accepts `path` (file path relative to the robot's working directory) and `behavior` (the name within the file). Both fields are required.

---

## 11.5 Conformance Requirements

- Only one behavior may run at a time. Running a new behavior MUST stop the current one first.
- All step executions MUST be logged with timestamp and outcome (§6 Invariant 3).
- A `stop` step MUST trigger an immediate motor e-stop (equivalent to §6 Invariant 1).
- The `GET /api/behavior/status` endpoint MUST be available whenever a behavior runtime is running.
- Implementations MUST gracefully handle a missing or invalid behavior file with a clear error response.
