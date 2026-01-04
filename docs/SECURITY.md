# Security Policy

This document describes the security posture, assumptions, and limitations of
HANS — Hardware-Aware Neural Storage.

HANS is a performance-oriented, cache-first storage system designed for AI
workloads. Security is important, but HANS intentionally makes **explicit trust
assumptions** in order to achieve its goals.

---

## Security Goals

HANS aims to:

- Operate safely under normal, non-malicious conditions
- Avoid crashes, data corruption, or hardware instability
- Fail safely and predictably when encountering errors
- Avoid introducing new attack surfaces beyond its scope

HANS does **not** attempt to provide comprehensive security guarantees beyond
these goals.

---

## Trust Model

HANS operates under the following trust assumptions:

### Trusted Environment

- The host operating system is trusted
- The kernel is trusted
- Device drivers (GPU, storage) are trusted
- HANS runs with appropriate system privileges

HANS is **not** designed to defend against a compromised kernel, malicious
drivers, or hostile hypervisors.

---

### Trusted Data Sources

HANS assumes that:

- Input data is **non-malicious**
- File formats are **well-formed**
- Persistent storage backends are trusted

HANS does **not** attempt to defend against:
- Malformed or adversarial file formats
- Deliberately corrupted datasets
- Malicious object store contents

This assumption is critical for format-aware optimizations.

---

### Trusted Users

HANS assumes:
- Users have legitimate access to the system
- Users do not attempt to exploit HANS to escalate privileges
- Multi-tenant isolation is handled externally

HANS is not a hardened multi-tenant isolation boundary.

---

## Threats Explicitly Out of Scope

HANS does **not** protect against:

- Adversarial datasets designed to exploit parsers
- Side-channel attacks (timing, power, cache)
- Denial-of-service via malicious workloads
- Data exfiltration via compromised clients
- GPU-level attacks or firmware exploits
- Speculative execution vulnerabilities

These threats are outside the design scope.

---

## Format-Aware Processing Safety

HANS supports format-aware optimizations for certain file types
(see `docs/FORMAT_AWARE.md`).

### Safety Guarantees

- Format parsing is **best-effort**
- Parsing failures immediately trigger fallback
- No unbounded parsing in hot paths
- No trust in user-provided metadata beyond validation

### Failure Behavior

If format-aware parsing fails:
- Structured access is aborted
- HANS falls back to opaque, whole-file I/O
- No retries occur in performance-critical paths

Format-aware features are optimizations, not requirements.

---

## GPU & Hardware Safety

HANS interacts with GPU hardware via standard vendor APIs
(see `docs/NVIDIA_INTEGRATION.md`).

### Safety Principles

- VRAM is treated as ephemeral
- Long-lived VRAM allocations are avoided
- All GPU interactions have fallback paths
- GPU errors trigger immediate degradation

### Failure Handling

If GPU telemetry or memory operations fail:
- GPU-aware features are disabled temporarily
- HANS reverts to CPU-only behavior
- Cache consistency is preserved

HANS prioritizes system stability over performance.

---

## Privilege Model

HANS may require elevated privileges for:

- eBPF telemetry
- FUSE mounting
- Low-level I/O optimizations

These privileges are expected to be granted explicitly by the operator.

HANS does not attempt privilege separation internally.

---

## Data Integrity

HANS is a **cache-first system**.

- Cached data may be lost at any time
- Cache metadata is best-effort
- Persistent storage is the source of truth

HANS does not provide:
- Strong consistency guarantees
- Cryptographic integrity checks
- End-to-end encryption

These concerns must be handled by underlying storage systems.

---

## Network Security

If HANS is deployed in a distributed configuration:

- Network security (TLS, authentication) is the responsibility of the operator
- HANS does not implement custom cryptographic protocols
- Trust boundaries must be enforced externally

---

## Reporting Security Issues

If you discover a security vulnerability in HANS:

1. **Do not open a public issue**
2. Contact the maintainer privately (method TBD)
3. Provide:
   - A clear description of the issue
   - Reproduction steps
   - Potential impact

Security reports will be handled responsibly.

---

## Responsible Disclosure

HANS follows responsible disclosure principles:

- Vulnerabilities will be assessed promptly
- Fixes will be prioritized based on severity
- Public disclosure will occur after mitigation where possible

Given the early stage of the project, timelines may vary.

---

## Summary

HANS is designed for **trusted environments running trusted workloads**.

By making its assumptions explicit, HANS avoids a false sense of security and
focuses on what it does best: delivering high-performance, hardware-aware
storage for AI workloads.

Operators are responsible for securing the environment in which HANS runs.
