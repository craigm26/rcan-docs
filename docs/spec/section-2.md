# §2 Role-Based Access Control

*Status: Stable · RCAN v2.1 · §2.8 M2M_PEER · §2.9 M2M_TRUSTED*

RCAN v2.1 defines a seven-level role hierarchy: GUEST (1), OPERATOR (2), CONTRIBUTOR (2.5), ADMIN (3), M2M_PEER (4), CREATOR (5), and M2M_TRUSTED (6). Higher roles inherit all permissions of lower roles. Roles are embedded in JWT tokens and enforced at the protocol layer before any command reaches the driver.

---

## 2.1 Overview

Access control in RCAN operates on two axes: **roles** (coarse-grained trust levels) and **scopes** (fine-grained permission grants). Both are carried in JWT tokens and validated before any message is dispatched.

Role inheritance is strict: a principal with role ADMIN also has all permissions of OPERATOR and GUEST. Roles are not additive across tokens; the highest role in the current session's token applies.

!!! warning ""
    **v2.1 breaking change:** The v1.x roles OWNER and LEASEE are removed. OWNER maps to ADMIN (3); LEASEE maps to OPERATOR (2). Update all JWT token issuers accordingly.

---

## 2.2 Role Hierarchy

| Role | Level | Rate Limit | Session TTL | Key Permissions |
|------|-------|-----------|------------|----------------|
| `M2M_TRUSTED` | 6 | Unlimited | 24 h max | Fleet orchestration: RRF-issued only, cross-fleet COMMAND/INVOKE/CONFIG, no safety bypass |
| `CREATOR` | 5 | Unlimited | None | Full hardware/software control, OTA push, safety override |
| `M2M_PEER` | 4 | Unlimited | None | Robot-to-robot: authorized by ADMIN, can issue COMMAND/INVOKE to fleet peers |
| `ADMIN` | 3 | 1000/min | 8 h | Configuration, skill installation, user management, M2M_PEER authorization |
| `OPERATOR` | 2 | 100/min | 2 h | Operational control within allowed modes |
| `CONTRIBUTOR` | 2.5 | 200/min | 4 h | Idle compute contribution scope only (CONTRIBUTE_REQUEST/RESULT/CANCEL) |
| `GUEST` | 1 | 10/min | 5 min | Status viewing, read-only telemetry, text chat |

!!! warning ""
    **TTL enforcement:** Control sessions expire per session TTL. Explicit renewal MUST be required before timeout. Implicit renewal MUST NOT occur. See [§6 Safety Invariants](section-6.md).

---

## 2.3 Scopes

Scopes are fine-grained permissions included in JWT tokens. A scope is only valid if the principal's role meets or exceeds the minimum role for that scope.

| Scope | Minimum Role | Grants |
|-------|-------------|--------|
| `status` | GUEST | Read health, telemetry, sensor data |
| `control` | OPERATOR | Issue commands, drive motors, run behaviors |
| `contribute` | CONTRIBUTOR | Donate idle compute: CONTRIBUTE_REQUEST, CONTRIBUTE_RESULT, CONTRIBUTE_CANCEL |
| `config` | ADMIN | Modify RCAN config, reload, OTA |
| `training` | ADMIN | Submit episodes, trigger learner, export data |
| `authority` | ADMIN | Respond to AUTHORITY_ACCESS (41) requests, FIRMWARE_ATTESTATION (43), SBOM_UPDATE (44) |
| `admin` | CREATOR | Safety overrides, hardware resets, key rotation |
| `fleet.trusted` | M2M_TRUSTED | Cross-fleet COMMAND/INVOKE/CONFIG without per-robot CREATOR approval. RRF-issued tokens only. |

---

## 2.4 Gateway Roles

Implementations MAY expose a simplified three-level gateway role system for human operators, mapped onto RCAN roles as follows:

| Gateway Role | RCAN Role | Default Scopes |
|-------------|----------|----------------|
| `admin` | ADMIN (3) | status, control, config, training |
| `operator` | OPERATOR (2) | status, control |
| `viewer` | GUEST (1) | status |

---

## 2.5 Conformance Requirements

- Implementations MUST enforce role-based rate limiting per source RURI.
- Session TTLs MUST be enforced. Expired sessions MUST be rejected with an `ERROR` response.
- Config changes (scope: `config`) MUST be issued by principals with ADMIN (level 3) or CREATOR (level 5) role. Any attempt from a lower role MUST be audited and rejected. See §16.3.
- Messages with `Priority.SAFETY` MUST skip rate-limiting queues (§6).
- M2M_PEER tokens MUST be issued by an ADMIN or CREATOR principal. Self-issued M2M tokens MUST be rejected.
- ESTOP messages from M2M_PEER principals MUST be honored — safety invariants apply to all role levels (§6).

---

## 2.6 Channel-Layer Scope Assignment (v1.6.2)

Implementations that receive commands via messaging channels (WhatsApp, Telegram, Slack, Signal, etc.) MUST assign an `rcan_scope` value to each inbound message at the channel boundary before passing the message to the command processing layer.

!!! info ""
    **Channel scope vs. JWT scope:** The `rcan_scope` assigned at the channel boundary is a coarse trust level for routing and rate-limiting purposes. It is distinct from the fine-grained JWT scopes in §2.3 (`status`, `control`, etc.), which are validated after authentication. Channel scope establishes the upper bound of what a sender may request; JWT scope enforces what an authenticated principal is granted.

### 2.6.1 Scope Resolution Rules

The assigned scope SHALL be determined by the sender's identity as follows, in priority order:

| Sender type | Assigned `rcan_scope` | Minimum LOA |
|------------|----------------------|------------|
| Owner / admin ID | `chat` | 1 |
| Allowlisted sender | `chat` | 0 |
| Robot peer (RRN pattern) | `status` | 0 |
| Pairing / unverified | `discover` | 0 |
| Unknown | Block (no scope) | — |

!!! warning ""
    Implementations MUST NOT assign a scope higher than `chat` to any channel-layer sender without explicit operator configuration (`rcan_protocol.channel_admin_scope`).

### 2.6.2 Scope Downgrade at API Boundary

The `/api/chat` endpoint (or equivalent inbound command handler) MUST enforce that `sender_scope` does not exceed `chat`. Any sender claiming a higher scope via channel metadata SHALL be silently downgraded to `chat` and the event logged.

### 2.6.3 Scope Metadata Field

Implementations SHOULD include `sender_scope` and `sender_loa` in the inbound message metadata object:

```json
{
  "sender_id":    "...",
  "sender_scope": "chat",
  "sender_loa":   0,
  "channel":      "whatsapp"
}
```

---

## 2.7 Multi-Hop Scope Propagation (v1.6.2)

When a robot delegates a task to a peer robot or sub-agent (R2R delegation), the originating command's scope MUST be preserved and propagated through the delegation chain.

!!! info ""
    **Relationship to §12 delegation chains:** Multi-hop scope propagation extends the `delegation_chain` model in §12 with an explicit non-escalation invariant. Each `DelegationHop` already carries a `scope` array; this section formalizes the ordering constraint and introduces `originating_scope` tracking at the envelope level.

### 2.7.1 Non-Escalation Invariant

For any delegation chain A → B → C:

```
scope_level(C's task) ≤ scope_level(B's task) ≤ scope_level(A's command)
```

Implementations MUST enforce this invariant at the delegation point. Scope demotion (lower scope than originating) is always permitted. Scope escalation is a protocol violation and MUST be rejected with a logged error.

The scope level ordering for this invariant is defined by the coarse channel-scope hierarchy (§2.6): `discover` < `status` < `chat` < `control` < `system` < `safety`.

### 2.7.2 Scope in Delegated Intent

When a robot issues a delegated command to a peer or sub-agent, the envelope MUST carry two scope fields alongside the existing `delegation_chain` (§12):

```json
{
  "intent_id":         "...",
  "originating_scope": "chat",
  "delegated_scope":   "chat",
  "delegation_chain":  []
}
```

`delegated_scope` MUST be ≤ `originating_scope` as defined by the scope level ordering in §2.6. Receivers MUST reject delegated commands where `delegated_scope` > `originating_scope`.

### 2.7.3 Agent Action Filtering

Robot implementations that use sub-agents to execute delegated tasks SHOULD enforce action-type filtering based on the delegated scope. The following is the reference action allowlist:

| Scope | Allowed action types |
|-------|---------------------|
| `discover` | ping, status, get_info |
| `status` | + get_telemetry, get_pose |
| `chat` | + speak, navigate_to, describe_scene |
| `control` | All actions |
| `system` | All actions |
| `safety` | All actions (including emergency stop) |

!!! warning ""
    Actions not in the allowlist for the current scope SHALL be vetoed by the safety/guardian layer. See [§6 Safety Invariants](section-6.md) for enforcement layer responsibilities.

---

## 2.8 M2M Authorization Roles (v2.1)

v2.1 introduces **M2M_PEER** (level 4) for machine-to-machine authorization. M2M_PEER allows one robot to issue `COMMAND` and `INVOKE` messages to fleet peers without a human principal in the loop — enabling autonomous multi-robot coordination.

### 2.8.1 Authorization Flow

M2M tokens are issued by a human ADMIN or CREATOR and scoped to a specific peer RRN:

```bash
# M2M_PEER authorization flow
# 1. An ADMIN issues an M2M authorization token for a specific peer robot
POST /api/m2m/authorize
{ "peer_rrn": "RRN-000000000005", "scopes": ["control", "status"], "expires_at": 1800 }

# 2. The peer robot carries this token in its JWT auth_token field
# 3. The receiving robot validates the M2M token against its ADMIN's public key
# 4. P66 safety invariant: M2M ESTOP is ALWAYS honored regardless of M2M role level
```

### 2.8.2 JWT Claims for M2M

M2M tokens carry a `rcan_role: "m2m_peer"` claim alongside the authorized scopes:

```json
{
  "sub":       "RRN-000000000005",
  "rcan_role": "m2m_peer",
  "rcan_scopes": ["control", "status"],
  "peer_rrn":  "RRN-000000000001",
  "exp":       1800,
  "iss":       "RRN-000000000001-admin"
}
```

### 2.8.3 M2M Safety Invariants

!!! danger ""
    **Safety invariant:** `Priority.SAFETY` (ESTOP) messages from M2M_PEER principals MUST be honored immediately. M2M role level does NOT gate safety messages. An M2M robot can always stop its peer — it cannot always command it.

---

## 2.9 M2M_TRUSTED Fleet Orchestration (v2.1)

**M2M_TRUSTED (level 6)** is a fleet-scope machine identity that sits above CREATOR (5) in the authorization hierarchy. Unlike M2M_PEER — which is authorized by a single ADMIN for peer-to-peer communication — M2M_TRUSTED is issued exclusively by the **Robot Registry Foundation (RRF)** root key and enables an orchestration system to command any robot in a registered fleet without requiring per-robot CREATOR approval at runtime.

The primary use case is a cloud management plane or on-premise fleet brain that needs to issue `COMMAND`, `INVOKE`, and `CONFIG` messages across an entire fleet atomically — e.g. a coordinated firmware rollout or multi-robot mission abort.

### 2.9.1 Why Above CREATOR?

M2M_TRUSTED sits at level 6 — above the human CREATOR — because fleet orchestration sometimes requires cross-cutting authority that no single robot's CREATOR can grant. A CREATOR can authorize access to *their* robot; they cannot authorize cross-fleet operations on robots owned by other CREATORs. M2M_TRUSTED tokens are issued by RRF after verifying *all* CREATORs in the fleet have registered their consent.

!!! danger ""
    **Hard constraint:** M2M_TRUSTED level does NOT grant safety bypass authority. `Priority.SAFETY` (ESTOP) signals from any source are always honored. A human CREATOR can always revoke an orchestrator's registration via RRF, taking effect within one revocation poll interval (default: 60 s).

### 2.9.2 Authorization Flow

```bash
# M2M_TRUSTED authorization flow
# 1. CREATOR registers an orchestration system with RRF, supplying its public key
POST https://api.rrf.rcan.dev/v2/orchestrators/register
{ "rrn": "RRN-000000000001", "orchestrator_key": "<ed25519-pubkey>", "justification": "Fleet brain" }

# 2. RRF validates the CREATOR's identity and issues a time-limited M2M_TRUSTED token
#    signed by the RRF root key (NOT by the CREATOR or any robot)
GET https://api.rrf.rcan.dev/v2/orchestrators/RRN-000000000001/token
→ { "token": "<jwt>", "rcan_role": "m2m_trusted", "scopes": ["fleet.trusted"], "exp": 86400 }

# 3. The orchestration system presents this token in every message envelope
# 4. Receiving robots verify the token against the RRF root public key
#    (published at https://api.rrf.rcan.dev/.well-known/rrf-root-pubkey.pem)

# HARD LIMITS — enforced in protocol, cannot be overridden by token claims:
# - ESTOP (Priority.SAFETY) from any principal is ALWAYS honored
# - M2M_TRUSTED may NOT issue tokens to other principals
# - M2M_TRUSTED tokens are scoped to a single registered orchestrator key
# - Token revocation is immediate: RRF revocation list is checked on every verify
```

### 2.9.3 JWT Claims for M2M_TRUSTED

```json
{
  "sub":         "orchestrator:fleet-brain-prod-001",
  "rcan_role":   "m2m_trusted",
  "rcan_scopes": ["fleet.trusted"],
  "fleet_rrns":  ["RRN-000000000001", "RRN-000000000005"],
  "exp":         86400,
  "iss":         "rrf.rcan.dev",
  "rrf_sig":     "base64url-ed25519-over-claims"
}
```

- `sub`: orchestrator identifier (not a robot RRN — orchestrators are not robots)
- `fleet_rrns`: explicit allowlist of robots this token can command; wildcards NOT permitted
- `exp`: maximum 24 h; must be re-issued daily
- `iss`: MUST be `rrf.rcan.dev`; any other issuer MUST be rejected
- `rrf_sig`: Ed25519 signature over the canonical JWT claims by the RRF root key

### 2.9.4 Hard Limits

- M2M_TRUSTED tokens MUST be issued by RRF only — self-issued tokens MUST be rejected.
- M2M_TRUSTED tokens MUST NOT be used to issue further tokens to other principals.
- M2M_TRUSTED tokens MUST be scoped to an explicit `fleet_rrns` list — no wildcard fleet access.
- M2M_TRUSTED tokens expire in ≤ 24 h; re-issuance requires re-validation by RRF.
- Revocation is near-real-time: receivers MUST poll the RRF revocation list at ≤ 60 s intervals when any M2M_TRUSTED session is active.
- ESTOP from any principal (including non-trusted ones) is always honored — level 6 does not gate safety signals.

### 2.9.5 Multi-Owner Consent

To register an orchestration system against a fleet that spans multiple CREATOR owners:

1. Each CREATOR submits a `CONSENT_GRANT (21)` message to RRF referencing the orchestrator key.
2. RRF aggregates consent; the token is only issued once all listed `fleet_rrns` CREATORs have consented.
3. Any CREATOR can revoke consent via `CONSENT_DENY (22)`; RRF revokes the token within 60 s.
