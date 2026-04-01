# Cylf

Cylf is a shared, format-agnostic codec layer for chunked data formats. It
provides a common foundation for encoding and decoding data chunks, so that
codec implementations, optimizations, and bug fixes benefit every format that
builds on it rather than being locked inside format-specific libraries.

Formats like Zarr, TIFF/COG, HDF5, Parquet, ORC, and others all use the same
fundamental codecs (zstd, lz4, shuffle, delta, bit-packing) but each
implements them independently. Cylf defines a shared interface for codecs, a
composition model for combining them into pipelines, and a runtime that
executes them, with a standard library of native codecs for performance and a
WebAssembly runtime for portability and extensibility.

---

## Documentation

### Start here

- [Motivation](01_motivation.md) — why a shared codec layer should exist, and
  what it enables
- [Architecture](02_architecture.md) — the components of the Cylf codec layer,
  how they relate, and where the boundaries lie

### Codec specifications

- [Signature Model](03_codecs/01_signature.md) — how codecs describe their
  interfaces: typed ports, bidirectional encode/decode blocks, awareness
  taxonomy, and format alias mappings
- [Pipeline Model](03_codecs/02_pipeline.md) — how codecs compose into
  executable DAGs: steps, wiring references, constants, explicit bidirectional
  definitions
- [Codec Inventory](03_codecs/03_inventory.md) — catalog of 60+ codecs across
  formats, with signatures, aliases, and composition properties

### Runtime specifications

- [Codec Contract](04_runtime/01_codec_contract.md) — the interface contract
  between the pipeline engine and codec implementations: WIT interface
  (Component Model), binary ABI (Core Wasm), native interface, and signature
  format
- [Distribution & Registry](04_runtime/02_distribution.md) — how codec
  artifacts are distributed: HTTPS, OCI registries, the warg ecosystem, native
  plugins, and embedded codecs

### Design decisions

- [Pipeline Architecture Tradeoffs](05_design/01_pipeline_arch.md) — analysis
  of mixed native + Wasm vs all-Wasm pipelines, Core module ABI vs Component
  Model, and how the two axes interact
- [Performance & Data Copy Accounting](05_design/02_perf_accounting.md) — copy
  counts per step boundary, the Canonical ABI bottleneck under Python, why a
  native orchestrator is required, and approaches to copy reduction
- [F3 Comparison](05_design/03_f3_comparison.md) — comparison with the F3 file
  format: where the approaches overlap, where they diverge, and how they could
  complement each other

### Roadmap

- [Open Questions & Roadmap](06_future.md) — unresolved design questions and
  planned work across the ecosystem

### Proposals

Forward-looking documents exploring how the codec layer could serve as
infrastructure for higher-level systems. These are drafts, not part of the
core spec.

- [Format Drivers and Data Orchestration](07_proposals/01_orchestration.md) —
  a plan format and execution model for orchestrating storage I/O, codec
  pipelines, format-specific callbacks, and concurrent chunk processing
- [CCRP and the Cylf Ecosystem](07_proposals/02_ccrp.md) — how the codec layer
  could enable a protocol for querying multidimensional datasets across formats
  and storage backends *(planned)*

---

## Implementations

Currenlly only one implementation exists:
[chonkle](https://github.com/cylf-dev/chonkle). This is a proof-of-concept
Python host for the codec pipeline engine. It is the first research
implementation intended to inform the spec and future development, not as the
reference runtime.

---

## Contributing

Cylf is an open source project at
[github.com/cylf-dev](https://github.com/cylf-dev). If you are interested in
contributing, the [Open Questions & Roadmap](06_future.md) lists areas where
input is most valuable.
