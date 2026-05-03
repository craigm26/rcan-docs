# §15 Swarm Coordination

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** A **swarm** is a named set of RCAN robots that can be queried and commanded together. Swarm membership is declared in a `swarm.yaml` node registry. Broadcast commands fan out to all nodes; each node enforces its own safety constraints independently.

---

## 15.1 Overview

Swarm coordination in RCAN is intentionally lightweight: there is no central coordinator, no consensus protocol, and no shared state. Each robot in the swarm operates independently; the swarm is simply the set of nodes a controller is authorized to address together.

This design prioritizes resilience: the failure of any single swarm node MUST NOT prevent other nodes from continuing to operate. Broadcast commands are fire-and-acknowledge; failures are reported per-node but do not abort the broadcast.

---

## 15.2 Node Registry Format

```yaml
# config/swarm.yaml — RCAN Swarm Node Registry
nodes:
  - name:  "alex"
    host:  "alex.local"
    ip:    "192.168.68.91"    # static IP fallback
    port:  8000
    token: "<OPENCASTOR_API_TOKEN for this node>"
    rcan:  "~/OpenCastor/alex.rcan.yaml"
    tags:  [rpi5, camera, i2c, rover]
    added: "2026-02-21"
```

### Node fields

| Field | Requirement | Description |
|-------|-------------|-------------|
| `name` | MUST | Human-readable alias for the node. |
| `host` | MUST | mDNS hostname or IP address of the robot. |
| `ip` | SHOULD | Static IP fallback when mDNS resolution fails. |
| `port` | SHOULD | RCAN protocol port (default: 8000). |
| `token` | MUST | API token (JWT or pre-shared key) for authenticating to this node. |
| `rcan` | MAY | Path to this node's `.rcan.yaml` config file. |
| `tags` | MAY | Array of tag strings for filtering in swarm queries. |
| `added` | MAY | ISO date when this node was added to the swarm. |

---

## 15.3 Swarm Operations

```json
// POST /api/command (broadcast to all swarm nodes)
// Implementors SHOULD forward to swarm peers when X-Swarm-Broadcast: true
{
  "instruction": "stop all motors",
  "broadcast":   true
}

// CLI shorthand
// castor swarm command --instruction "go forward" [--node alex]
// castor swarm stop   [--node alex]
// castor swarm status [--json]
```

The `X-Swarm-Broadcast: true` HTTP header signals that a command should be forwarded to all swarm peers. Implementations SHOULD support this header on the `POST /api/command` endpoint. The `broadcast: true` field in the JSON body is an equivalent signal for WebSocket transport.

---

## 15.4 Required Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET  /api/status` | Per-node health (includes ruri, version, caps). |
| `POST /api/command + X-Swarm-Broadcast: true` | Fan-out command to all swarm peers. |
| `POST /api/stop` | Immediate e-stop; MUST respond within 500 ms. |

---

## 15.5 Safety Rules

- **E-stop is always local:** A swarm broadcast MUST NOT override a node's local e-stop state. If a node is in e-stop, it MUST acknowledge the broadcast but MUST NOT execute the command.
- **Stop latency:** `POST /api/stop` MUST respond within 500 ms from all nodes. Implementations SHOULD respond with motor stop confirmation, not just HTTP acknowledgement.
- **Partial failure:** If a broadcast reaches N nodes and M < N fail, the response SHOULD include per-node status. The caller is responsible for deciding whether partial execution is acceptable.
- **Auth scope:** Swarm broadcasts require the same scope as the underlying command type. A broadcast COMMAND requires `control` scope on all target nodes.
