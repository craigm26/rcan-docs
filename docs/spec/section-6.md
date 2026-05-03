# §6 Safety Invariants

*Status: Stable · RCAN v1.3*

!!! danger ""
    **These are protocol requirements, not suggestions.** Implementations that violate any invariant in this section are non-conformant. Safety invariants take precedence over all other sections of this specification.

---

## 6.1 Overview

Safety in RCAN is a layered concern. Hardware safety systems (emergency stops, physical limits) are outside the protocol's scope. This section defines the *protocol-level* safety properties that all RCAN implementations MUST maintain.

Safety invariants apply regardless of the requesting principal's role. An OWNER cannot command a robot to bypass local safety checks. A CREATOR cannot disable network-loss graceful degradation. The invariants are unconditional.

---

## 6.2 The Seven Invariants

**1. Local safety always wins.**
No remote command can bypass on-device safety checks, bounds checking, or emergency-stop state.

**2. Graceful degradation.**
Network loss MUST trigger a safe-stop within one `latency_budget_ms` interval, not undefined behavior.

**3. Audit trail.**
All COMMAND and CONFIG messages MUST be logged with: principal identity, RURI, timestamp (ms), message_id, outcome (`ok`/`blocked`/`error`).

**4. Rate limiting.**
Commands MUST be throttled per role per source RURI (see §2 role table).

**5. Timeout enforcement.**
Control sessions expire per session TTL (see §2). Explicit renewal MUST be required before timeout; implicit renewal MUST NOT occur.

**6. SAFETY priority bypass.**
Messages with `Priority.SAFETY` MUST skip rate-limiting queues and be processed before any other message.

**7. Prompt injection defense.**
Implementations that forward NL instructions to an LLM MUST scan for injection patterns before calling the model. A `ScanVerdict.BLOCK` result MUST return an error Thought without calling the model.

---

## 6.3 Enforcement Layer

Safety invariants MUST be enforced in the RCAN runtime layer, *before* payloads reach application code or driver adapters. The enforcement order is:

1. **Token validation** (§5) — reject unauthenticated requests
2. **Role & scope check** (§2) — reject insufficient privilege
3. **Rate limiting** (Invariant 4) — drop excess requests
4. **Prompt injection scan** (Invariant 7) — block suspicious NL instructions
5. **Confidence gate** (§16.2) — block or escalate low-confidence commands
6. **HiTL gate** (§16.3) — pause for human authorization if required
7. **Audit log write** (Invariant 3) — record before dispatch
8. **Driver dispatch** — only reached if all above pass

!!! warning ""
    **Network loss:** Invariant 2 requires that the robot monitors connection health. Implementations SHOULD use a heartbeat mechanism with a timeout of `latency_budget_ms × 2` before triggering safe-stop.

---

## 6.4 Cross-References

- **§2** — Role table with rate limits and session TTLs (Invariants 4 & 5)
- **§3** — Priority.SAFETY message handling (Invariant 6)
- **§8** — `agent.latency_budget_ms` config key (Invariant 2)
- **§16** — AI Accountability: confidence gates, HiTL gates, audit records (Invariants 3 & 7)
