# Appendix B: WebSocket Transport Binding

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** This appendix specifies how RCAN messages are carried over a WebSocket connection (RFC 6455). It covers frame format, the mandatory CONNECT handshake, heartbeat requirements, reconnection strategy, sequence number conventions, and WS-level error codes.

---

## B.1 Framing

RCAN messages are transmitted as **WebSocket text frames** containing UTF-8 encoded JSON. Each WebSocket frame carries exactly one RCAN message object.

### Text Frames (Required)

- All RCAN control messages (CONNECT, STATUS, COMMAND, INVOKE, etc.) MUST use JSON text frames.
- Message objects MUST be valid JSON (RFC 8259).
- Implementations MUST NOT split a single RCAN message across multiple WebSocket frames.

### Binary Frames (Optional)

Binary frames MAY be used for high-frequency telemetry streams where JSON overhead is prohibitive. Supported encodings:

- **Protobuf** — Protocol Buffers (binary serialisation). Message type MUST be identified via a preceding text-frame envelope or an agreed session-level type negotiation.
- **MessagePack** — Compact binary JSON-compatible encoding. Field names MUST match the RCAN JSON field names exactly.

Servers that do not support binary frames MUST close the connection with code `1003` (Unsupported Data) upon receiving a binary frame.

!!! warning ""
    **Note:** Binary frame support is optional. Implementations that require guaranteed interoperability SHOULD use JSON text frames exclusively.

---

## B.2 Handshake

After the WebSocket connection is established (HTTP Upgrade complete), the client MUST send a CONNECT message as its very first frame.

```json
// Client → Server (first message after WS open)
{"type":"CONNECT","ruri":"rcan://robot.local","version":"1.3","caps":{"move":{"version":"1.0"}}}

// Server → Client (success)
{"type":"CONNECT_ACK","session_id":"sess_abc123","server_version":"1.3"}

// Server → Client (failure — closes WS with code 4001)
{"type":"ERROR","code":8001,"name":"ConnectionRefused","message":"Auth token missing"}
```

### Rules

1. The client MUST send CONNECT as the first frame. Any other message type sent first MUST be rejected by the server with a close frame (code `4001`).
2. The server MUST respond with either `CONNECT_ACK` (success) or close the WebSocket with code `4001` (ConnectionRefused) and an ERROR payload.
3. No other messages may be exchanged until CONNECT_ACK is received.
4. The server MAY impose a handshake timeout (RECOMMENDED: 10 seconds). If CONNECT is not received in time, the server MUST close with code `1002`.

| WS Close Code | Meaning |
|---------------|---------|
| `4001` | ConnectionRefused — authentication failed, version mismatch, or server at capacity |
| `1002` | Protocol error — non-CONNECT first frame, or malformed JSON |

---

## B.3 Heartbeat

RCAN uses an application-level heartbeat (PING/PONG) in addition to, or instead of, the WebSocket protocol-level ping/pong frames.

```json
// Client sends every 30 seconds
{"type":"PING","msg_id":"ping_001","timestamp_us":1741737600000000}

// Server responds within 5 seconds
{"type":"PONG","reply_to":"ping_001","timestamp_us":1741737600005000}
```

### Rules

1. The client MUST send a PING message every **30 seconds** of inactivity (no other messages sent).
2. The server MUST respond with a PONG within **5 seconds** of receiving a PING.
3. If the client receives no PONG within 5 seconds of sending a PING, it MUST close the connection with code `1001` (Going Away) and initiate reconnect (see B.4).
4. If the server receives no PING for more than **60 seconds**, it MAY close the connection with code `1001`.

!!! info ""
    **Tip:** In practice, STATUS messages sent by the robot implicitly act as keep-alive signals. Clients SHOULD reset the PING timer on receipt of any message from the server.

---

## B.4 Reconnect Strategy

Clients MUST implement exponential backoff when reconnecting after a connection loss. A new CONNECT message MUST be sent on each reconnect attempt.

```text
// Reconnect backoff schedule (±500 ms jitter applied at each step)
Attempt 1: wait  1 s   →  reconnect
Attempt 2: wait  2 s   →  reconnect
Attempt 3: wait  4 s   →  reconnect
Attempt 4: wait  8 s   →  reconnect
Attempt 5: wait 16 s   →  reconnect
Attempt 6+: wait 30 s  →  reconnect (cap)
```

### Backoff Parameters

| Parameter | Value |
|-----------|-------|
| Initial wait | 1 second |
| Multiplier | ×2 per attempt |
| Maximum wait (cap) | 30 seconds |
| Jitter | ±500 ms (uniform random, applied after cap) |

### Rules

1. Clients MUST reset the backoff counter to the initial value (1 s) after a successful CONNECT_ACK.
2. Clients MUST re-send a full CONNECT message (including `caps`) on each reconnect. Servers MUST NOT assume session state persists across connections.
3. If the server responds to CONNECT with code `4002` (AuthExpired), the client MUST refresh its auth token before retrying and MUST NOT use the backoff schedule for this specific case — it MUST wait until a new token is available.
4. Clients SHOULD surface persistent reconnect failures to the user or operator after 5 consecutive failed attempts.

---

## B.5 Sequence Numbers

Messages SHOULD include a monotonically increasing `seq` field (integer). Sequence numbers help receivers detect dropped messages and reorder out-of-order delivery.

```json
{"type":"STATUS","seq":1042,"ruri":"rcan://robot.local"}
```

### Rules

1. Senders SHOULD include `seq` in STATUS, COMMAND, INVOKE, and INVOKE_RESULT messages.
2. Sequence numbers SHOULD start at 1 and increment by 1 per message. They MAY wrap to 1 after reaching `2^31 − 1`.
3. Receivers MAY use `seq` gaps to detect dropped messages and emit a warning, but MUST NOT block processing waiting for a missing sequence number.
4. Sequence numbers are session-scoped — they reset on reconnect.

!!! info ""
    **Note:** WebSocket provides ordered, reliable delivery on a single connection. Sequence numbers are primarily useful for detecting message loss at the application layer (e.g. after reconnect) and for correlating telemetry samples across multiple streams.

---

## B.6 Error Codes

The following error codes are defined for the WebSocket Transport Binding. RCAN error codes in the 8xxx range map to specific WebSocket close codes.

| RCAN Code | Name | WS Close Code | Meaning |
|-----------|------|---------------|---------|
| `8001` | `ConnectionRefused` | `4001` | Server refused the connection — auth failure, version incompatibility, or server at capacity. Client SHOULD display an error and MAY retry with backoff. |
| `8002` | `AuthExpired` | `4002` | The session auth token has expired. Client MUST refresh the token before reconnecting. Do NOT use the standard backoff schedule. |
| `8003` | `HeartbeatTimeout` | `1001` | No PONG received within the heartbeat window (B.3). The connection is considered dead. Client MUST reconnect with backoff. |
| `8004` | `MessageTooLarge` | `1009` | An inbound message exceeded the server's maximum frame size. The connection is closed. Client SHOULD reduce payload size before reconnecting. |
| `8005` | `InvalidFrame` | `1007` | A text frame contained invalid UTF-8 or a malformed JSON object. The connection is closed. Client SHOULD fix message serialisation before reconnecting. |

### Mapping Summary

```
8001 → WS 4001
8002 → WS 4002
8003 → WS 1001
8004 → WS 1009
8005 → WS 1007
```
