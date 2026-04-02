# Design Decisions

Analysis and rationale behind key architectural choices in Cylf.

- [Pipeline Architecture Tradeoffs](01_pipeline_arch.md) — analysis of mixed
  native + Wasm vs all-Wasm pipelines, Core module ABI vs Component Model, and
  how the two axes interact
- [Performance & Data Copy Accounting](02_perf_accounting.md) — copy counts per
  step boundary, the Canonical ABI bottleneck under Python, why a native
  orchestrator is required, and approaches to copy reduction
- [F3 Comparison](03_f3_comparison.md) — comparison with the F3 file format:
  where the approaches overlap, where they diverge, and how they could
  complement each other
