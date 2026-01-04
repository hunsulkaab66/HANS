# HANS Policy Engine

This document defines the **Policy Engine** for HANS — Hardware-Aware Neural
Storage.

The Policy Engine is the decision-making core of HANS. It consumes telemetry,
workload context, and system constraints, and produces **concrete actions**
that govern caching, placement, prefetching, eviction, and throttling behavior.

Without the Policy Engine, HANS is just a fast cache.
With it, HANS becomes AI-native.

---

## Purpose

The Policy Engine exists to answer a single question:

> Given current hardware state, workload behavior, and system constraints,
> what is the best action to take *right now*?

Actions include:
- Where data should be placed
- Whether to prefetch or stall
- Which data to evict
- Whether to throttle background activity
- How aggressively to use hardware resources

---

## Design Principles

The Policy Engine is built on the following principles:

1. **Telemetry-Driven, Not Static**
   - Policies respond to live signals.
   - Static configuration is only a starting point.

2. **Fail Closed, Fallback Fast**
   - If any signal is unavailable or inconsistent, HANS immediately falls back
     to conservative, opaque I/O behavior.

3. **Edge-Safe by Default**
   - Policies must never crash the GPU or system.
   - Performance is secondary to stability on edge devices.

4. **Deterministic, Not Magical**
   - Decisions must be explainable.
   - No hidden heuristics or implicit behavior.

5. **Composable Policy Domains**
   - GPU, power, format, and cache policies are independent inputs.
   - The Policy Engine arbitrates conflicts explicitly.

---
<!--
## High-Level Architecture

+-------------------+
Telemetry Inputs
GPU (NVML)
Memory Pressure
I/O Latency
Power / Thermal
Access Patterns
+---------+---------+
      |
      v
+-------------------+
Policy Engine
Signal Normalizer
Rule Evaluator
Conflict Resolver
Action Generator
+---------+---------+
      |
      v
+-------------------+
Action Targets
Cache Manager
Tier Manager
Prefetch Engine
Eviction Engine
IO Engine
+-------------------+


The Policy Engine **does not perform I/O**.
It issues decisions to other components.

---
-->
## Inputs

### 1. Telemetry Signals

Collected continuously via the `TelemetryEngine`.

#### GPU Signals
- GPU utilization
- Free and total VRAM
- Active kernel count
- Memory allocation failures
- PCIe / NVLink throughput (if available)

Source: NVML, CUDA runtime

---

#### Memory Signals
- Free system RAM
- Pinned memory usage
- Allocation latency
- Fragmentation indicators

Source: OS metrics, allocator telemetry

---

#### I/O Signals
- Read/write latency
- Queue depth
- io_uring completion lag
- Backing store responsiveness

---

#### Power & Thermal Signals
- Power draw
- Thermal zone temperatures
- Throttling events
- Power mode (edge profiles)

Source: sysfs, platform APIs

---

### 2. Workload Context

Provided by the CacheManager and client interfaces.

- Job ID / workload ID
- Job type (training, inference, feature lookup)
- Priority class
- Dataset identity
- Access pattern classification (sequential, random, strided)
- Epoch or iteration hints (if available)

---

### 3. Static Configuration

- Deployment profile (edge vs datacenter)
- Maximum cache sizes per tier
- Safety thresholds
- Feature enablement flags

Static configuration **never overrides safety signals**.

---

## Policy Domains

The Policy Engine evaluates decisions across multiple policy domains.

Each domain produces **constraints and recommendations**, not final actions.

---

### 1. GPU Policy Domain

Defined in detail in `NVIDIA_INTEGRATION.md`.

Responsibilities:
- Prevent VRAM overcommit
- Minimize GPU stalls
- Avoid long-lived VRAM allocations
- Prefer pinned host memory

Example outputs:
- VRAM admission allowed / denied
- Maximum chunk size for GPU staging
- Prefetch aggressiveness limits
- Forced eviction from VRAM staging buffers

---

### 2. Power & Thermal Policy Domain

Defined in `POWER_THERMAL.md`.

Responsibilities:
- Prevent thermal throttling
- Respect power budgets
- Reduce background activity when constrained

Example outputs:
- Throttle background persistence
- Reduce prefetch depth
- Pause speculative reads
- Enforce low-power mode

---

### 3. Format-Aware Policy Domain

Defined in `FORMAT_AWARE.md`.

Responsibilities:
- Determine semantic access opportunities
- Validate safe parsing boundaries
- Decide between opaque vs structured access

Example outputs:
- Chunk alignment recommendations
- Selective read permissions
- Format-aware prefetch hints
- Fallback to opaque access

---

### 4. Cache & Tier Policy Domain

Core storage behavior.

Responsibilities:
- Decide tier placement
- Enforce tier capacity constraints
- Select eviction candidates

Example outputs:
- Preferred tier for new data
- Eviction priority scores
- Pinning recommendations
- Spillover targets

---

## Conflict Resolution

Policy domains may produce conflicting recommendations.

Example:
- GPU policy wants smaller chunks
- Format policy wants larger aligned reads

The Policy Engine resolves conflicts using **explicit priority rules**.

---

### Priority Order (Default)

1. **Safety (GPU, Power, Thermal)**
2. **Correctness**
3. **Stability**
4. **Performance**
5. **Efficiency**

Safety constraints always win.

---

### Conflict Resolution Strategy

1. Apply hard constraints (must-not-violate)
2. Intersect soft recommendations
3. If no feasible solution exists:
   - Fall back to conservative defaults
   - Log the decision path

No policy is allowed to force unsafe behavior.

---

## Action Generation

Once constraints are resolved, the Policy Engine emits actions.

### Action Types

- `PLACE(chunk, tier)`
- `EVICT(chunk)`
- `PREFETCH(chunk, depth)`
- `THROTTLE(component)`
- `STALL(request)`
- `FALLBACK(mode)`

Actions are idempotent and reversible.

---

## Timing Model

The Policy Engine operates on **multiple time scales**.

### Fast Path (Per Request)
- Admission control
- Chunk sizing
- Tier selection

Must be lightweight and non-blocking.

---

### Medium Path (Periodic)
- Prefetch tuning
- Eviction scoring
- Tier pressure balancing

Runs on a fixed interval.

---

### Slow Path (Reactive)
- Power mode changes
- Thermal events
- GPU fault recovery

Triggered by events.

---

## Failure Semantics

### Signal Loss

If telemetry is unavailable or inconsistent:
- Mark signal as degraded
- Reduce policy aggressiveness
- Prefer opaque, synchronous behavior

---

### Parsing Failures

If format-aware parsing fails:
- Abort structured access
- Fallback to whole-file I/O
- Never retry parsing in hot paths

---

### GPU Errors

If GPU memory allocation fails:
- Immediately free staging buffers
- Disable VRAM usage temporarily
- Notify CacheManager

---

## Observability & Explainability

Every policy decision should be traceable.

The Policy Engine must expose:
- Decision logs
- Active constraints
- Policy overrides
- Degradation reasons

This is critical for debugging and trust.

---

## Non-Goals

The Policy Engine explicitly does NOT:
- Use machine learning for decisions (initially)
- Perform speculative learning across reboots
- Persist complex policy state
- Guarantee global optimality

Correctness and safety come first.

---

## Future Extensions

Possible future enhancements:
- ML-assisted policy tuning
- Cross-node cooperative policies
- Workload-specific policy plugins
- User-provided policy hints

These must not compromise determinism or safety.

---

## Summary

The Policy Engine is the brain of HANS.

By cleanly separating telemetry, policy domains, conflict resolution, and action
generation, HANS can evolve intelligently without becoming unpredictable or
unsafe.

All optimization flows through policy.
