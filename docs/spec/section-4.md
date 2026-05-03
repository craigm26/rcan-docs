# §4 Discovery (mDNS)

*Status: Stable · RCAN v1.3*

For LAN discovery without cloud connectivity, implementations MUST broadcast a `_rcan._tcp.local.` mDNS service record. TXT records carry the robot's RURI, capabilities, and status — enabling zero-configuration discovery on local networks.

---

## 4.1 Overview

RCAN discovery is built on Multicast DNS (mDNS) and DNS-SD (RFC 6762 / RFC 6763). Any RCAN-capable device on a LAN MUST advertise itself so that clients can discover robots without manual configuration or cloud connectivity.

Discovery is complementary to registry-based addressing (§1). Even when a robot has a canonical RURI with a cloud registry, it SHOULD also broadcast its mDNS record so LAN clients can discover it without internet access. This is the "Local Supremacy" principle from [§7 Federation](section-7.md).

---

## 4.2 Service Type

Implementations MUST register a DNS-SD service of type:

```
_rcan._tcp.local.
```

The instance name SHOULD be the human-readable robot name (e.g. `Living Room Companion._rcan._tcp.local.`). The SRV record's port MUST match `rcan_protocol.port` from the robot's config (default: 8000).

DNS-SD browse query:

```
dns-sd -B _rcan._tcp local
```

---

## 4.3 Required TXT Records

```yaml
ruri=rcan://continuon.cloud/continuon/companion-v1/d3a4b5c6
model=companion-v1
caps=nav,vision,chat,teleop,arm,status
roles=owner,user,guest
version=1.3.0
name=Living Room Companion
status=idle
```

| Key | Requirement | Description |
|-----|-------------|-------------|
| `ruri` | MUST | Canonical RURI of the robot (§1) |
| `model` | MUST | Robot model identifier |
| `caps` | MUST | Comma-separated list of standard capability names (§9) |
| `roles` | SHOULD | Comma-separated list of roles accepted (owner, user, guest) |
| `version` | SHOULD | RCAN protocol version (semver) |
| `name` | MAY | Human-readable robot name for display in discovery UIs |
| `status` | SHOULD | One of: idle, running, paused, error |
| `npu` | MAY | NPU/accelerator identifier (e.g. hailo-8l, coral-tpu). Null if none. (v1.7) |
| `tops` | MAY | Peak NPU throughput in TOPS (e.g. 26). Omit if no NPU. (v1.7) |

!!! info ""
    **caps field:** The `caps` value MUST be a comma-separated list of standard capability names as defined in [§9](section-9.md). Custom capabilities MUST use reverse-domain notation (e.g. `com.acme.gripper`).

---

## 4.4 Conformance Requirements

- Implementations MUST broadcast `_rcan._tcp.local.` when `rcan_protocol.enable_mdns: true`.
- The `ruri` TXT record MUST be present and valid (§1 RURI format).
- The `caps` TXT record MUST be present and reflect active capabilities.
- The `status` TXT record SHOULD be updated within 5 seconds of a state change.
- On shutdown, the robot SHOULD send a mDNS goodbye packet (TTL=0) to remove the record from caches.
- Clients MUST handle robots that omit optional TXT fields gracefully.
