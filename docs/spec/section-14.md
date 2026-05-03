# §14 Provider Management

*Status: Stable · RCAN v1.3*

!!! info ""
    **Overview:** An RCAN robot runtime manages one or more LLM "brains" and MUST handle provider failures gracefully. Two fallback strategies are defined: quota fallback (for billing/rate-limit errors) and offline fallback (for network loss).

---

## 14.1 Overview

LLM providers are external dependencies that can fail for various reasons: quota exhaustion, rate limiting, network outages, or service disruptions. RCAN defines protocol-level resilience strategies so that robots continue to operate when their primary provider is unavailable.

Both fallback strategies are transparent to the controller: the robot continues to accept commands and produce responses. The telemetry stream (§13) exposes `provider` and `using_fallback` fields so monitoring systems can observe provider state.

---

## 14.2 Quota Fallback (`provider_fallback`)

When the primary provider returns a quota or billing error (HTTP 402/429 or keywords: *credits exhausted*, *rate limit*, *quota*), the runtime MUST:

1. Switch transparently to the configured `provider_fallback` provider.
2. Record the switch timestamp in the audit log.
3. Alert the operator via the configured `alert_channel`.
4. After `quota_cooldown_s` seconds, attempt to restore the primary provider on the next request.

!!! warning ""
    **Detection:** Implementations MUST detect quota errors both by HTTP status code (402, 429) and by case-insensitive substring matching on the error message body. Not all providers return consistent status codes for quota exhaustion.

---

## 14.3 Offline Fallback (`offline_fallback`)

When the runtime detects internet loss (via HTTP reachability check), it MUST switch to a local provider (Ollama, llama.cpp, MLX, etc.) automatically. The switch back to cloud occurs after connectivity is restored and verified.

- Reachability checks SHOULD be performed every `check_interval_s` seconds.
- A single failed check SHOULD NOT trigger fallback — implementations SHOULD require 2–3 consecutive failures.
- After switching back to the primary provider, the runtime MUST re-check connectivity before each request until 5 consecutive successful checks have been recorded.

---

## 14.4 Config Blocks

```yaml
provider_fallback:
  enabled:          true
  provider:         "ollama"        # target fallback provider
  model:            "llama3.2:3b"
  quota_cooldown_s: 3600            # seconds before retrying primary
  alert_channel:    "telegram"      # channel to notify on switch

offline_fallback:
  enabled:         true
  provider:        "ollama"
  model:           "llama3.2:3b"
  check_interval_s: 30
  alert_channel:   "telegram"
```

---

## 14.5 Health Check Interface

All provider adapters MUST implement a `health_check()` method:

```typescript
health_check() → { "ok": bool, "latency_ms": float, "error": str | null }
```

- The runtime SHOULD call `health_check()` on the fallback provider at startup.
- The result MUST be surfaced at `GET /api/provider/health`.
- A failed health check on the fallback provider at startup SHOULD log a warning but MUST NOT prevent the runtime from starting (the primary may still be healthy).
- The `latency_ms` field provides round-trip inference latency for monitoring purposes.
