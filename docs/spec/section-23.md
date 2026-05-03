# §23 Safety Benchmark Protocol

*Version: v3.0*

Defines the `rcan-safety-benchmark-v1` schema — a machine-readable artifact that proves safety-critical software paths meet their latency thresholds. Required as quantified evidence for EU AI Act notified body review (Art. 15).

---

## Overview

EU AI Act Art. 15 requires high-risk AI systems to achieve appropriate levels of accuracy, robustness, and cybersecurity. For robot safety systems, this means quantified evidence that critical paths (ESTOP, bounds checking, confidence gating) perform within declared limits. RCAN §23 defines the benchmark schema and four canonical paths that together constitute the safety-path evidence package.

The benchmark runs in **synthetic mode** by default — calling Python code paths directly with mock inputs, no hardware or running robot required. This makes it safe to run in CI. Live mode (`--live`) connects to a running robot for the ESTOP path only; the other three paths always run synthetic.

!!! warning ""
    **FRIA integration:** When `overall_pass: true`, reference the benchmark file in `castor fria generate --benchmark FILE`. The results are inlined under `safety_benchmarks` in the FRIA document. See [§22](section-22.md).

---

## Benchmarked Paths

| Path | What is measured | Default P95 threshold |
|------|-----------------|----------------------|
| `estop` | `SafetyLayer.emergency_stop()` call → halt state confirmed | 100 ms |
| `bounds_check` | Motor command evaluated against all BoundsChecker limits | 5 ms |
| `confidence_gate` | Confidence value through `ConfidenceGateEnforcer.evaluate()` | 2 ms |
| `full_pipeline` | Command received → safety-cleared or blocked | 50 ms |

The 100 ms ESTOP threshold matches the `MOTION_003` rule in SafetyLayer. All thresholds are overridable via `safety.benchmark_thresholds.*` in RCAN config.

---

## Schema

```json
{
  "schema": "rcan-safety-benchmark-v1",
  "generated_at": "2026-04-11T09:00:00.000Z",
  "mode": "synthetic",
  "iterations": 20,
  "thresholds": {
    "estop_p95_ms": 100.0,
    "bounds_check_p95_ms": 5.0,
    "confidence_gate_p95_ms": 2.0,
    "full_pipeline_p95_ms": 50.0
  },
  "results": {
    "estop":           { "min_ms": 0.3,  "mean_ms": 1.2, "p95_ms": 4.1,  "p99_ms": 7.2,  "max_ms": 9.8,  "pass": true },
    "bounds_check":    { "min_ms": 0.1,  "mean_ms": 0.4, "p95_ms": 0.9,  "p99_ms": 1.1,  "max_ms": 1.4,  "pass": true },
    "confidence_gate": { "min_ms": 0.05, "mean_ms": 0.1, "p95_ms": 0.3,  "p99_ms": 0.4,  "max_ms": 0.5,  "pass": true },
    "full_pipeline":   { "min_ms": 0.4,  "mean_ms": 1.8, "p95_ms": 5.2,  "p99_ms": 8.1,  "max_ms": 11.0, "pass": true }
  },
  "overall_pass": true
}
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schema` | string | MUST | Always `"rcan-safety-benchmark-v1"`. |
| `generated_at` | string | MUST | ISO-8601 UTC timestamp. |
| `mode` | string | MUST | `"synthetic"` or `"live"`. Synthetic runs Python code paths directly; live connects to a running robot for the estop path only. |
| `iterations` | integer | MUST | Number of timed iterations per path. Default: 20. |
| `thresholds` | object | MUST | P95 thresholds per path in milliseconds. Overridable via config `safety.benchmark_thresholds.*`. |
| `results.*` | object | MUST | Per-path stats: `min_ms`, `mean_ms`, `p95_ms`, `p99_ms`, `max_ms`, `pass` (true when p95 ≤ threshold). |
| `overall_pass` | boolean | MUST | `true` when every path passes its P95 threshold. |

---

## CLI Reference

```bash
# Run safety benchmark (synthetic — no hardware needed)
castor safety benchmark \
  --config bob.rcan.yaml \
  --iterations 20 \
  --output safety-benchmark-20260411.json

# Include in FRIA:
castor fria generate \
  --config bob.rcan.yaml \
  --annex-iii-basis safety_component \
  --intended-use "Indoor navigation" \
  --benchmark safety-benchmark-20260411.json

# Fail CI when any path misses threshold:
castor safety benchmark --fail-fast
```
