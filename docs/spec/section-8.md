# §8 Robot Config (RCAN File)

*Status: Stable · RCAN v1.3*

A RCAN config file (extension `.rcan.yaml`) is the primary artifact that describes a robot. It defines identity, hardware drivers, LLM agent configuration, network settings, and capability declarations. Implementations MUST validate configs against the published JSON Schema before starting any runtime.

---

## 8.1 Overview

The `.rcan.yaml` file is the single source of truth for a robot's configuration. It is human-readable, version-controlled, and schema-validated. One robot = one config file.

Implementations MUST validate configs against the JSON Schema published at [rcan.dev/schema/rcan.schema.json](https://rcan.dev/schema/rcan.schema.json) before starting any runtime. A config that fails schema validation MUST prevent the runtime from starting.

---

## 8.2 Required Top-Level Keys

| Key | Type | Description |
|-----|------|-------------|
| `rcan_version` | string (semver) | Spec version this config targets |
| `metadata` | object | Identity: robot_name, robot_uuid, author, license |
| `agent` | object | LLM provider, model, latency budget, safety flags |
| `physics` | object | Kinematic type, DoF, chassis dimensions |
| `drivers` | array (≥1) | Hardware driver configurations |
| `network` | object | Telemetry and remote access flags |
| `rcan_protocol` | object | Port, capabilities, mDNS, JWT flags |

---

## 8.3 Reference Example

```yaml
rcan_version: "1.3.0"

metadata:
  robot_name:   "Alex"
  robot_uuid:   "550e8400-e29b-41d4-a716-446655440000"
  author:       "craigm26"
  license:      "Apache-2.0"
  manufacturer: "opencastor"
  model:        "rover"
  tags:         [rpi5, camera, i2c, rover]

agent:
  provider:          "huggingface"
  model:             "Qwen/Qwen2.5-VL-7B-Instruct"
  latency_budget_ms: 3000
  safety_stop:       true

provider_fallback:
  enabled:          true
  provider:         "ollama"
  model:            "llama3.2:3b"
  quota_cooldown_s: 3600
  alert_channel:    "telegram"

physics:
  type: "differential_drive"
  dof:  2
  wheel_circumference_m:  0.21
  turn_time_per_deg_s:    0.008

drivers:
  - protocol: "pca9685"
    port:     "/dev/i2c-1"
    address:  "0x40"
    channels:
      left_motor:  0
      right_motor: 1

camera:
  type:          "oakd"
  resolution:    [640, 480]
  framerate:     30
  depth_enabled: true

rcan_protocol:
  port:         8000
  capabilities: [nav, vision, chat, teleop, status]
  enable_mdns:  true
  enable_jwt:   true
```

---

## 8.4 AI Gate Configuration (v1.2)

The `agent` block accepts two optional keys introduced in v1.2 for AI-driven robots: `confidence_gates` and `hitl_gates`. Both are arrays of gate objects evaluated at command dispatch time. See [§16](section-16.md) for full behavioral specification.

```yaml
agent:
  confidence_gates:
    - scope: control
      min_confidence: 0.5
      on_fail: escalate   # escalate | block | allow
    - scope: nav
      min_confidence: 0.7
      on_fail: block
    - scope: admin
      min_confidence: 0.95
      on_fail: block
  hitl_gates:
    - action_types: [arm_deploy]
      require_auth: true
      auth_timeout_ms: 30000
      on_timeout: block   # block | allow
      notify: [whatsapp]
```

---

## 8.5 Optional Blocks

The full JSON Schema includes the following optional config blocks. These extend the robot's capabilities without requiring changes to the required keys:

- `tiered_brain` — multi-layer LLM stack (reactive / fast / planner)
- `reactive` — rule-based pre-filter for common actions
- `learner` — online learning and episode recording
- `camera` — camera type, resolution, framerate, depth
- `audio` — microphone and TTS configuration
- `channels` — notification channels (telegram, whatsapp, etc.)
- `swarm` — swarm membership and coordination (§15)
- `ros2` — ROS2 bridge configuration
- `provider_fallback` — quota fallback provider (§14)
- `offline_fallback` — offline fallback provider (§14)

!!! info ""
    **Schema reference:** Full JSON Schema at [rcan.dev/schema/rcan.schema.json](https://rcan.dev/schema/rcan.schema.json)

---

## 8.6 Multi-Runtime Agent Declaration (v3.2)

As of v3.2, the `agent` block accepts an optional `runtimes` array. Each entry declares one runtime that can boot the robot. This enables peer-runtime deployments where a single ROBOT.md identity is valid under multiple agent harnesses (e.g. `robot-md` and `opencastor`) without modifying the manifest or the RRF registry record.

```yaml
agent:
  runtimes:
    - id: robot-md
      harness: claude-code
      default: true
      models:
        - provider: anthropic
          model: claude-sonnet-4-6
          role: primary
        - provider: anthropic
          model: claude-opus-4-7
          role: planning
      safety_stop: true
      latency_budget_ms: 200
    - id: opencastor
      harness: castor-default
      models:
        - provider: anthropic
          model: claude-sonnet-4-6
          role: primary
        - provider: google
          model: gemini-er-1.6
          role: manipulation
      recipes: [pick-place-basic]
      safety_stop: true
      latency_budget_ms: 150
```

### Field Rules

| Key | Type | Description |
|-----|------|-------------|
| `runtimes[].id` | string (required) | Runtime identifier. Known: "robot-md", "opencastor". Community-extensible. |
| `runtimes[].harness` | string (required) | Runtime-specific harness name. Interpretation belongs to the named runtime. |
| `runtimes[].default` | boolean (optional) | If runtimes[] has multiple entries, exactly one MUST be default: true. |
| `runtimes[].models` | array (optional) | Model declarations. Shape and semantics are runtime-specific. |

Unknown per-entry fields are preserved (runtime-specific pass-through) — implementations MUST NOT reject entries that carry fields they don't recognize. Common agent-level keys (`safety_stop`, `latency_budget_ms`, and similar) MAY appear inside a `runtimes[]` entry to override the top-level `agent` defaults for that runtime only.

### Backward Compatibility

The flat `agent.provider` / `agent.model` form (introduced in v1.2) remains valid in v3.2. Validators SHOULD emit a `DeprecationWarning` when the flat form is encountered and SHOULD normalize it internally to a single-entry `runtimes[]`. The flat form is scheduled for removal in v4.0.

### Known Runtime IDs

- `robot-md` — prebuilt agent harnesses (Claude Code and peers). [pypi.org/project/robot-md](https://pypi.org/project/robot-md/)
- `opencastor` — custom harness with swappable models and recipes. [pypi.org/project/opencastor](https://pypi.org/project/opencastor/)

Third-party runtimes may define their own ids. See the peer-runtimes listing in the `opencastor-ops` repository for criteria and current entries.

---

## 8.7 Voice Block (v3.3, optional)

As of v3.3, ROBOT.md MAY declare a top-level `voice` block that travels with the robot's manifest. Pendant implementations and other voice-aware hosts read it to discover wake-word aliases beyond the robot's `name`, the spoken language, and an advisory TTS voice id. The block is optional and additive — robots that omit it MUST still function under hosts that respect `name` as the primary wake word.

```yaml
voice:
  aliases: [bobby, hey-bob]   # extra wake words beyond `name`
  language: en-US             # BCP-47, default "en-US"
  tts_voice: en_US-amy-low    # Piper voice id, advisory hint
```

### Field Rules

| Key | Type | Description |
|-----|------|-------------|
| `voice.aliases` | array\<string\> (optional) | Extra wake-word aliases beyond the robot's `name`. Matched case-insensitively after Unicode-NFKC normalization. Empty array means "no extras"; omitted means the same. |
| `voice.language` | string (optional) | BCP-47 language tag for spoken interaction. Default "en-US". Validators SHOULD warn (not reject) on malformed tags. |
| `voice.tts_voice` | string (optional) | Advisory TTS voice id (e.g. Piper voice). Hosts MAY ignore or substitute. Not part of cryptographic identity — never affects RRN, signing, or compliance artifacts. |

### Normalization

Hosts MUST normalize each entry in `voice.aliases` and the robot's `name` via Unicode NFKC + case-fold before wake-word matching. Hosts MAY additionally strip ASCII punctuation. The spec does not define a canonical alias string for cryptographic comparison — wake matching is fuzzy by nature.

### Validation

Validators SHOULD emit a *warning* (not a rejection) when:

- `voice.language` is malformed under BCP-47.
- `voice.aliases` contains the robot's own `name` (redundant — already a wake word).
- `voice.aliases` contains entries that NFKC-collapse to the same string.

### Out of Scope

`voice.tts_voice` is an advisory hint only. It is NOT part of the canonical manifest pre-image for any RCAN signing, RRN derivation, or §22-26 compliance artifact. Hosts that don't recognize the voice id MUST fall back to a default voice without error.
