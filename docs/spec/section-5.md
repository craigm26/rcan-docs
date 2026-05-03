# §5 Authentication

*Status: Stable · RCAN v1.3*

RCAN uses JWT (JSON Web Tokens) for authentication. Two token types are defined: a full RCAN JWT (carrying role, scope, fleet claims) and a simplified gateway JWT for human operators. All tokens MUST be validated before any message is dispatched.

---

## 5.1 Overview

Authentication in RCAN is token-based. The `auth_token` field in the message envelope (§3) carries a JWT. Robots MUST validate the token before processing any message type that requires a scope. `DISCOVER` messages are the only type exempt from token validation.

When `rcan_protocol.enable_jwt: true` is set in the robot config, all non-DISCOVER messages without a valid token MUST be rejected with an `ERROR` response.

---

## 5.2 RCAN JWT (Device-Level)

Full RCAN tokens include all identity, role, scope, and fleet claims:

```json
{
  "sub":   "550e8400-e29b-41d4-a716-446655440000",
  "iss":   "rcan://continuon.cloud/continuon/companion-v1/d3a4b5c6",
  "aud":   "rcan://continuon.cloud/continuon/companion-v1/*",
  "role":  "owner",
  "scope": ["status", "control", "config", "training"],
  "fleet": ["d3a4b5c6", "a1b2c3d4"],
  "exp":   1735689600,
  "iat":   1735603200
}
```

### Required claims

| Claim | Requirement | Description |
|-------|-------------|-------------|
| `sub` | MUST | Subject — UUID v4 of the principal (device or user) |
| `iss` | MUST | Issuer — RURI of the issuing robot or gateway |
| `aud` | MUST | Audience — RURI (or wildcard) of the target robot |
| `role` | MUST | RCAN role string: owner, leasee, user, guest, creator |
| `scope` | MUST | Array of granted scopes (status, control, config, training, admin) |
| `exp` | MUST | Expiry time (Unix seconds) |
| `iat` | MUST | Issued-at time (Unix seconds) |
| `fleet` | SHOULD | Array of device-id values this token is valid for |

---

## 5.3 Gateway JWT (Operator-Level)

Implementations MAY also issue simplified gateway tokens for human operators. These MUST be mapped to a RCAN role before authorization decisions are made:

```json
{
  "sub":  "alice",
  "role": "operator",
  "iss":  "opencastor-gateway",
  "exp":  1735689600,
  "iat":  1735603200
}

/* Gateway roles map to RCAN roles:
   admin    → OWNER  (level 4)
   operator → LEASEE (level 3)
   viewer   → GUEST  (level 1)  */
```

---

## 5.4 Verification Order

Implementations MUST validate tokens in this order:

1. Validate JWT signature (HMAC-SHA256 minimum; RS256 recommended for production)
2. Check `exp` and `iat` claims
3. Verify `aud` matches the robot's RURI (wildcard matching allowed — §1.5)
4. Map `role` to RCAN level; check level ≥ required for requested scope
5. If `fleet` is present, verify target device-id is in the list

!!! warning ""
    **Algorithm note:** HMAC-SHA256 is acceptable for development and LAN deployments. Production systems with remote principals MUST use RS256 (asymmetric) to prevent shared-secret leakage across untrusted networks.

---

## 5.5 Token Endpoint

Implementations SHOULD expose a token endpoint for gateway authentication:

```http
POST /auth/token
Body:  { "username": "alice", "password": "..." }
Reply: { "access_token": "...", "token_type": "bearer", "role": "operator" }
```

The response token MUST be a valid JWT carrying the `role` mapped from the gateway role system. The `token_type` field MUST be `"bearer"`.
