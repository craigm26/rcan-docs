# §7 Federation

*Status: Stable · RCAN v1.3*

RCAN is **federated like email**, not centralised like a platform. Any organisation can run its own registry. Robots can migrate between registries. Local discovery always works, even if the cloud registry is unavailable.

---

## 7.1 Overview

Federation means that no single organisation controls the RCAN namespace. The `rcan://` scheme is open; anyone can operate a registry at their own domain. Robots are identified by their full RURI including the registry domain — just as email addresses are identified by their domain.

This design prevents vendor lock-in: a robot purchased from one manufacturer can be migrated to a community registry without hardware changes. The robot's local capabilities remain intact regardless of what happens to any upstream registry.

---

## 7.2 Registry Model

RCAN supports multiple co-existing registries:

| RURI Prefix | Description |
|------------|-------------|
| `rcan://continuon.cloud/...` | Managed service (Continuon cloud registry) |
| `rcan://my-local-server.lan/...` | Self-hosted registry on a private LAN server |
| `rcan://robotics-co-op.org/...` | Community-run registry |

The `local.rcan` pseudo-registry is reserved for LAN-only deployments. RURIs with `local.rcan` MUST NOT be routed to any external network; they are valid only within the local broadcast domain.

---

## 7.3 The Right to Redirect

> **The Right to Redirect:** An OWNER (level 4) can always change the robot's upstream registry without losing local discovery.

This means:

- Changing the `rcan_protocol.registry` config key is a valid OWNER operation.
- After a registry change, mDNS broadcast continues with the new RURI (§4).
- Existing session tokens from the old registry MUST be invalidated after the change.
- No third party (including the previous registry operator) can prevent an OWNER from migrating.

---

## 7.4 Local Supremacy

> **Local Supremacy:** Even if the cloud registry deletes your ID, `_rcan._tcp.local` discovery always works.

The robot's mDNS advertisement (§4) operates independently of any cloud registry. A robot that is removed from a cloud registry continues to be discoverable on the LAN and continues to operate normally. Cloud deregistration MUST NOT disable local functionality.

This also applies to network outages: an RCAN robot MUST degrade gracefully on internet loss ([§6](section-6.md) Invariant 2) and continue to accept local commands from LAN principals with valid local-registry tokens.

---

## 7.5 Cross-References

- **§1** — RURI canonical form and local.rcan pseudo-registry
- **§4** — mDNS discovery (local supremacy mechanism)
- **§6** — Graceful degradation on network loss (Safety Invariant 2)
- **§21** — Robot Registry Integration: RRN↔RURI canonical mapping and registry handshake
