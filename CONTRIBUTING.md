# Contributing to HANS

Thank you for your interest in contributing to **HANS — Hardware-Aware Neural Storage**.

HANS is an early-stage, performance-driven open-source project focused on building an AI-native, hardware-aware storage cache for modern AI workloads.

Contributions are welcome — but HANS has a strong architectural vision and explicit scope. Please read this document carefully before contributing.

---

## Project Status

🚧 **HANS is currently in pre-alpha**

- APIs are unstable
- Major refactors are expected
- Backward compatibility is not guaranteed and unlikely
- Documentation may lag behind implementation

Early contributions should prioritize **correctness, clarity, and benchmarks**
over polish.

---

## Guiding Principles

All contributions should align with the following principles:

1. **Serve the Canonical Workload**
   - See `docs/CANONICAL_WORKLOAD.md`
   - If a change does not benefit the canonical workload, it likely does not belong in HANS.

2. **Hardware Awareness First**
   - GPU, VRAM, memory pressure, and I/O characteristics are first-class inputs.
   - Avoid abstractions that hide hardware state.

3. **Performance Is a Feature**
   - Code clarity matters, but performance regressions are unacceptable.
   - Benchmarks matter more than micro-optimizations.

4. **Edge Constraints Are Real**
   - Edge devices are not “small datacenters.”
   - Designs must handle tight VRAM, power, and thermal limits gracefully.

5. **Simplicity Over Generality**
   - HANS is not a general-purpose filesystem.
   - Avoid adding features “just in case.”

---

## How to Contribute

### 1. Start With an Issue

Before submitting a pull request:

- Open an issue describing:
  - The problem you want to solve
  - How it relates to HANS goals
  - Any proposed design or trade-offs

This helps avoid duplicated work and misaligned contributions.

---

### 2. Pick a Contribution Type

Good early contributions include:

- Bug fixes
- Performance improvements
- Benchmarks and tooling
- Documentation improvements
- Refactoring for clarity (with justification)

Please avoid large architectural changes without prior discussion.

---

### 3. Development Workflow

1. Fork the repository
2. Create a feature branch
3. Make small, focused commits
4. Include tests or benchmarks where applicable
5. Open a pull request with a clear description

---

## Coding Guidelines

### General
- Prefer explicit code over clever abstractions
- Avoid unnecessary dependencies
- Keep hot paths simple and measurable
- Comment non-obvious performance decisions

### C++ / Systems Code
- Avoid hidden allocations in hot paths
- Be explicit about ownership and lifetimes
- Prefer RAII where appropriate
- Measure before and after changes

### GPU-Related Code
- Avoid long-lived VRAM allocations
- Assume VRAM is scarce and volatile
- Always consider fallback paths when GPU features are unavailable

---

## Benchmarks & Performance Changes

If your change affects performance:

- Include benchmark results where possible
- State:
  - Hardware used
  - Dataset size
  - Cache configuration
- Clearly indicate improvements or regressions

Performance-related PRs without measurements may be rejected.

---

## Testing Expectations

- New functionality should include tests when feasible
- Bug fixes should include regression tests
- Benchmarks should be reproducible

Early-stage exceptions are acceptable, but correctness is non-negotiable.

---

## Documentation Contributions

>[!IMPORTANT]
>Documentation is highly valued.
>
>Good documentation contributions include:
>- Clarifying architecture
>- Explaining trade-offs
>- Updating outdated assumptions
>- Adding diagrams or examples
>
>Docs should remain honest about limitations.


---

## Code of Conduct

HANS follows a simple rule:

> Be respectful, constructive, and professional.

<mark>Harassment, hostility, or dismissive behavior will not be tolerated.</mark>

---

## Licensing

By contributing to HANS, you agree that your contributions will be licensed under the **Apache License 2.0**, consistent with the rest of the project.

---

## Final Note

HANS is opinionated by design.

Not every good idea belongs in this project — and that’s intentional.
If you’re excited about building fast, hardware-aware systems for AI workloads, you’re very welcome here.

Thank you for contributing!!! 
