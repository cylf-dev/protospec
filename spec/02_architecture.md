# Architecture

Cylf is a shared codec layer for chunked data formats. It provides a
format-agnostic foundation for encoding and decoding data chunks, so that
codec implementations, optimizations, and bug fixes benefit every format
that builds on it rather than being locked inside format-specific libraries.

This document describes the conceptual architecture: what the components are,
how they relate, and where the boundaries lie. For detailed specifications, see
the linked reference documents. For the reasoning behind key design choices, see
the [Design Decisions](05_design/README.md) documents.

---

## The Problem Cylf Addresses

Every chunked data format re-implements the same codec algorithms independently,
multiplying maintenance burden across formats, languages, and codecs.

Cylf addresses this by defining a shared interface for codecs and a composition
model for combining them into pipelines, independent of any particular format.
A format library that builds on Cylf delegates encoding and decoding to the
shared layer. When a codec is improved, added, or fixed in Cylf, every format
that uses it benefits without per-format integration work.

See [Motivation](01_motivation.md) for a more detailed framing.

---

## Ecosystem Diagram

```plain
  ┌──────────────────────────────────────────────────────────┐
  │                   Format Drivers                         │
  │                                                          │
  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │
  │   │  Zarr    │  │  TIFF/   │  │ Parquet  │  │  HDF5   │  │
  │   │  driver  │  │  COG     │  │  driver  │  │ driver  │  │
  │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬─────┘  │
  └────────┼─────────────┼─────────────┼────────────┼────────┘
           │             │             │            │
           │  Reads format-specific file metadata   │
           │  and constructs pipeline definitions:  │
           │  which codecs, in what order, with     │
           │  what parameters.         │            │
           │             │             │            │
           ▼             ▼             ▼            ▼
  ┌──────────────────────────────────────────────────────────┐
  │                  Cylf Codec Layer                        │
  │                                                          │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │              Pipeline Engine                       │  │
  │  │                                                    │  │
  │  │  Receives pipeline definitions. Resolves each      │  │
  │  │  step to a codec implementation. Validates codec   │  │
  │  │  signatures against pipeline requirements.         │  │
  │  │  Executes the DAG. Handles fan-out, fan-in,        │  │
  │  │  dependency ordering.                              │  │
  │  └───────────────────────┬────────────────────────────┘  │
  │                          │                               │
  │              resolves codecs from                        │
  │                          │                               │
  │            ┌─────────────┴─────────────┐                 │
  │            ▼                           ▼                 │
  │  ┌──────────────────┐   ┌─────────────────────────────┐  │
  │  │ Standard Library │   │     WASM Runtime            │  │
  │  │                  │   │                             │  │
  │  │ Native compiled  │   │  Portable modules fetched   │  │
  │  │ implementations  │   │  on demand from registries  │  │
  │  │ of common codecs │   │  (OCI, HTTPS, warg) and     │  │
  │  │                  │   │  cached locally             │  │
  │  │ zstd, lz4, zlib, │   │                             │  │
  │  │ shuffle, delta,  │   │  Custom encodings,          │  │
  │  │ bitpack, ...     │   │  format-specific variants,  │  │
  │  │                  │   │  new algorithms             │  │
  │  └──────────────────┘   └─────────────────────────────┘  │
  │                                                          │
  │  Every codec — whether from the standard library or a    │
  │  WASM module — carries its own typed signature. The      │
  │  pipeline engine discovers signatures at load time and   │
  │  validates them against the pipeline's requirements.     │
  └──────────────────────────────────────────────────────────┘
```

A **format driver** reads format-specific file metadata (TIFF IFDs, Zarr
`.zarray` JSON, Parquet file footers, HDF5 superblock and object headers) and
constructs a **pipeline definition**, the DAG of codec steps needed to decode
(or encode) a chunk, along with the parameters those codecs require. The format
driver is the boundary between format-specific knowledge and the format-agnostic
codec layer. Cylf does not define format drivers; they are the responsibility
of format libraries. Cylf does define the interface they target: codec signatures
and the pipeline schema.

---

## Components

### Codec Signatures

Every codec — whether a native standard library implementation or a WASM
module — carries its own **signature**: a typed description of its interface
in each direction it supports, with separate `encode` and `decode` blocks
declaring named input and output ports. Signatures are intrinsic to the codec;
the pipeline engine discovers them at load time and validates them against the
pipeline's requirements before execution begins.

Each codec has a **canonical identifier** (e.g. `zstd`, `shuffle`,
`tiff-predictor-2`) and is classified by **awareness** — the minimum knowledge
it needs about the data beyond raw bytes, from pure-bytes (opaque byte streams)
through structure-aware (requiring dimensional or format-specific layout). The
awareness classification determines what parameters a format driver must extract
from format metadata and wire into the pipeline.

The [Codec Inventory](03_codecs/03_inventory.md) is a reference catalog of known
codecs — their signatures, format aliases, awareness classifications, and
pipeline decompositions. It is a specification resource, not a runtime component.

- Full specification: [Codec Signature Model](03_codecs/01_signature.md)
- Reference catalog: [Codec Inventory](03_codecs/03_inventory.md)

### Pipeline Model

A **pipeline definition** is a directed acyclic graph (DAG) of codec steps
that describes how to decode (or encode) a chunk's data.

The DAG structure supports arbitrary composition, including fan-out, fan-in,
and named port routing, so format drivers can express multi-stream codec logic
(e.g. Parquet page splits) naturally rather than implementing custom
orchestration. Each direction (encode and decode) is defined independently with
its own steps and wiring. A pipeline is itself a codec: its interface has the
same structure as a leaf codec signature.

- Full specification: [Pipeline Model](03_codecs/02_pipeline.md)

### Pipeline Engine

The **pipeline engine** receives a pipeline definition, resolves each step to a
codec implementation, validates codec signatures against the pipeline's
requirements, determines execution order via topological sort of the DAG, and
runs each step, routing data between steps according to the wiring references.

The engine treats all codec implementations uniformly. A pipeline step
references a codec by its canonical identifier; the engine resolves it to
either a standard library implementation or a WASM module. The choice is
invisible to the pipeline definition and to the format driver.

The engine resolves codecs from two sources:

**The standard library** contains native compiled implementations of widely-used
codecs: the pure-bytes compressors (zstd, lz4, deflate, gzip, bzip2, snappy,
brotli), common pre-compression transforms (shuffle, bitshuffle, delta), and
other codecs where performance is critical and the algorithm is well-established.
These run at full native speed with no sandbox overhead. The standard library is
the pragmatic core: the codecs that most formats need most of the time.

**The WASM runtime** executes codecs distributed as portable WebAssembly
modules. A WASM codec is a Component Model component implementing a defined
interface (`encode` and `decode` functions operating on named port maps). WASM
codecs are:

- **Portable**: the same `.wasm` binary runs on any OS and architecture. The
  runtime compiles it to native machine code on first use and caches the result.
- **Sandboxed**: a codec can only access its own memory. It cannot touch the host
  process, the filesystem, or the network unless explicitly permitted.
- **Distributable**: modules can be fetched from registries (OCI artifacts,
  HTTPS, or the emerging WASM component registry ecosystem) on demand, cached
  locally, and verified via signatures.
- **Language-agnostic**: any language with a WASM Component Model toolchain
  (Rust, C/C++, Python, Go) can produce a conforming codec.

The WASM runtime is the extensibility mechanism. Data producers can adopt new
codecs, format-specific variants, or experimental compression algorithms without
requiring consumers to update their client libraries. As long as the codec
module can be fetched and executed by the WASM runtime, any consumer with a
Cylf-based toolchain can decode the data — no per-codec library updates, no
dependency chasing, no platform-specific builds.

- Runtime specification: [Codec Contract](04_runtime/01_codec_contract.md)
- Distribution strategy: [Distribution & Registry](04_runtime/02_distribution.md)
- Design rationale: [Why WASM](05_design/01_pipeline_arch.md),
  [Why Component Model](05_design/01_pipeline_arch.md)

---

## Boundaries

### What Cylf is

Cylf is a codec layer. It defines how codecs are described (signatures), how
they compose (pipeline definitions), and how they execute (the pipeline engine
with its standard library and WASM runtime). It provides native implementations
of common codecs and an extensibility mechanism (WASM) for everything else.

### What Cylf is not

Cylf is not a file format. It does not define how data is laid out in files
or object stores, how files are indexed, or how datasets are partitioned. Those
are the responsibilities of formats and their drivers.

Cylf is not a query engine. It does not parse queries, plan execution, or manage
I/O scheduling. Those are the responsibilities of the systems that sit above
format drivers.

Cylf is not a data access protocol. It does not define how chunks are located,
requested, or transferred over a network. However, the codec layer is designed
to be a foundation that such protocols can build on.
See [RFC: CCRP and the Cylf Ecosystem](07_proposals/02_ccrp.md) for one exploration of
this direction.

### The format driver boundary

Format drivers are described in the [Ecosystem Diagram](#ecosystem-diagram)
section above. Cylf defines the interface that format drivers target via codec
signatures and the pipeline schema, but Cylf does not prescribe how format
drivers are implemented.

The long-term possibility of Cylf-based format drivers that handle data
orchestration (reading, decoding, and assembling chunks into composite output
structures, or the inverse for writing) is explored in [RFC: Format Drivers and
Data Orchestration](07_proposals/01_orchestration.md).

---

## Further Reading

**Specifications:**
- [Codec Signature Model](03_codecs/01_signature.md)
- [Pipeline Model](03_codecs/02_pipeline.md)
- [Codec Inventory](03_codecs/03_inventory.md)
- [Codec Contract](04_runtime/01_codec_contract.md) (WIT interface)
- [Distribution & Registry](04_runtime/02_distribution.md)

**Design Decisions:**
- [Pipeline Architecture Tradeoffs](05_design/01_pipeline_arch.md) (WASM, Component Model, mixed vs all-WASM)
- [Performance & Data Copy Accounting](05_design/02_perf_accounting.md)
- [Comparison with F3](05_design/03_f3_comparison.md)

**Forward-Looking:**
- [RFC: Format Drivers and Data Orchestration](07_proposals/01_orchestration.md)
- [RFC: CCRP and the Cylf Ecosystem](07_proposals/02_ccrp.md)

**Implementation:**
- [chonkle](https://github.com/cylf-dev/chonkle) — Proof-of-concept Python
  host for the codec runtime and pipeline engine. Research implementation; not
  intended as the reference runtime.

**Open Questions:**
- [Open Questions & Roadmap](06_future.md)
