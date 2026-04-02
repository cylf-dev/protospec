# Codec Pipeline Architecture: Trade-off Analysis

This document analyzes the trade-offs across two related design axes for codec
pipelines that support WebAssembly:

1. **Mixed native + Wasm codecs vs. all-Wasm codecs** — whether to allow
   non-Wasm codecs (shared libraries, C extensions, etc.) alongside Wasm
   codecs in the same execution graph.

2. **Core module ABI vs. Component Model vs. both** — which Wasm interface style
   to use for the Wasm codecs: raw integer pointer/length conventions (Core), the
   typed WIT/Canonical ABI layer (Component Model), or a mix of the two.

These axes interact. The combination chosen affects performance, portability,
security, testability, and operational complexity. Both axes are analyzed
separately, then the interactions are summarized.

## Background: Data Movement at Step Boundaries

Understanding copy counts is foundational to both analyses. Wasm's sandbox model
means a module can read and write only its own linear memory — it cannot
dereference a host pointer. The host can read and write the module's linear
memory but not the reverse. This isolation is why data copies at step boundaries
are currently unavoidable.

For a Component Model codec invocation, two copies occur at each step boundary
regardless of host language — one to lower data into the component's linear
memory, one to lift output back to the host:

```
     host bytes
         │
      copy in   (lower: host → component linear memory)
         │
         ▼
component linear memory
         │
    codec runs
         │
         ▼
component linear memory
         │
      copy out  (lift: component linear memory → host)
         │
         ▼
     host bytes
```

Core Wasm codecs differ: the host can access the module's linear memory
directly. When the next step is also a Core Wasm codec, the orchestrator can
transfer data between linear memories with a single `memcpy` — one copy total
for the edge, not two.

The number of copies per edge, and their cost, depend on both source and
destination backend:

| Step A → Step B                               | Copies | Transfer mechanism            | Speed (Python host)  |
|-----------------------------------------------|--------|-------------------------------|----------------------|
| Native → Native                               | 0      | In-process reference          | —                    |
| Native → Core module                          | 1      | `Memory.write()` (ctypes)     | ~10 GB/s             |
| Core module → Native                          | 1      | `Memory.read()` (ctypes)      | ~10 GB/s             |
| Core module → Core module (Python host)       | 1      | `ctypes.memmove`              | ~10 GB/s             |
| Component → Component (Python host)           | 2      | Canonical ABI lift + lower    | 1.4–1.9 MB/s †       |
| Component → Component (native host)           | 2      | Single memcpy per direction   | ~10 GB/s             |

† The Python binding (`wasmtime._component._types.ListType`) iterates every byte
individually in Python: one `isinstance()` check and one ctypes field write per
byte. At 2 MB, this is ~2 million Python loop iterations per copy direction. The
Rust binding for the same Wasm binary does a single `memory.write()` call (one
memcpy). Measured throughput: 1.7 MB/s (Python) vs. 9,113 MB/s (Rust) at
identical buffer sizes. This bottleneck is in the Python binding layer; it is not
a property of the Component Model architecture. The correct fix is a native
(Rust or C) orchestrator.

## Part 1: Mixed Native + Wasm vs. All-Wasm Pipelines

### Definitions

A **native codec** runs directly in the host process: shared libraries, C
extensions, or language-native implementations. No sandbox, no runtime overhead,
full access to host memory and hardware.

A **Wasm codec** is compiled to a `.wasm` binary and executed inside a Wasm
runtime (e.g., Wasmtime). The runtime's sandbox enforces memory isolation.

> Note: `.wasm` files are not directly executable. The runtime compiles them to
> native machine code on first load and can cache the result on disk.
> The `.wasm` artifact is portable; the compiled output is platform-specific.

### Mixed Native and Wasm Pipelines

**Pros**

- **Incremental adoption.** Existing native codecs can be used immediately
  without rewriting. Teams can introduce Wasm codecs alongside existing
  libraries without a full migration.

- **Zero-copy at native-to-native boundaries.** When two adjacent pipeline steps
  are both native, data passes as an in-process reference. No serialization or
  memory copy occurs at that edge.

- **Full hardware access.** Native codecs have unrestricted access to GPUs,
  hardware crypto accelerators, architecture-specific SIMD (AVX, NEON), and any
  OS-level facility. Wasm does not support GPU offload and relies on WASI for
  system interfaces, which does not yet cover all hardware access patterns.

- **Mature toolchain.** Native codecs use standard package managers and standard
  build tools. No Wasm-specific toolchain is required for the native steps.

- **Native debugging.** Stack traces from native codecs use standard frames.
  Debuggers and profilers attach directly without Wasm frame indirection.

**Cons**

- **Non-uniform interface contract.** Native and Wasm codecs have different
  calling conventions. The orchestrator must implement at least two code paths.

- **Inconsistent security boundary.** Wasm codecs are sandboxed; native codecs
  run in the host process without isolation. A bug or supply-chain compromise in
  a native codec has full access to the host process, its memory, and its
  filesystem. The overall pipeline's security posture is determined by the
  least-sandboxed codec.

- **Heterogeneous distribution.** Native codecs are platform-specific builds.
  Wasm codecs are `.wasm` binaries with different registries and verification
  schemes. Versioning, pinning, auditing, and caching use two distinct
  mechanisms.

- **Portability limited by native codecs.** A pipeline is only as portable as its
  least-portable native codec.

- **No static composition.** `wasm-tools compose` and `wac` (the WebAssembly
  composition tools) operate only on Component Model components. A pipeline that
  includes native steps cannot be statically merged into a single artifact.

### All-Wasm Pipelines

**Pros**

- **Single orchestrator code path.** The orchestrator deals only with Wasm
  codecs; there is no native dispatch branch to maintain in parallel.

- **Portable artifacts.** The same `.wasm` binary runs on any OS and architecture
  that the runtime supports without changes to the artifact.

- **Uniform security boundary.** Every codec is sandboxed. A buggy or compromised
  codec cannot access host memory, the filesystem, or the network beyond what the
  runtime explicitly grants via WASI.

- **Uniform distribution.** All codecs are `.wasm` artifacts. They use the same
  registry and caching infrastructure.

- **Language-agnostic authoring.** Any language with a Wasm toolchain — Rust
  (`cargo-component`), C/C++ (`wit-bindgen-c`), Python (`componentize-py`), Go
  (`wit-bindgen-go`) — can implement the same codec interface.

- **Static composition (Component Model, fixed topology).** `wasm-tools compose`
  can merge a fixed pipeline topology into a single component. Intra-component
  calls share a single linear memory, so inter-step copies are eliminated.

**Cons**

- **At least 1 copy per step boundary.** Wasm's sandbox requires data to cross
  the host–codec boundary at each step edge. Component Model codecs incur 2
  copies per edge. Core Wasm codecs incur 1 copy per edge for adjacent Core
  steps. Neither matches the 0-copy path available when two native codecs share
  process memory.

- **Canonical ABI overhead under Python orchestrators (Component Model only).**
  With wasmtime-py, each copy direction through the Canonical ABI runs at
  1.4–1.9 MB/s for `list<u8>` data. A native orchestrator eliminates this: both
  copy directions run at memcpy speed (~10 GB/s).

- **Limited hardware access.** GPU compute, architecture-specific SIMD beyond
  what the Wasm SIMD proposal covers, and OS-level I/O not exposed through WASI
  are not available inside a codec.

- **Toolchain requirements for codec authors.** Codec authors need a
  Wasm-capable toolchain. Component Model toolchains vary in maturity by
  language.

- **Runtime scope.** Full Component Model support requires Wasmtime or
  Wasmer 4+. Browser-native Component Model support requires polyfills.
  IoT-targeted runtimes have limited or no Component Model support.

- **Debugging requires runtime-aware tools.** Useful stack traces from Wasm
  frames require DWARF debug info and runtime support for source-mapped frames.

### Summary: Mixed vs. All-Wasm

| Property                    | Mixed Native + Wasm                  | All-Wasm                              |
|-----------------------------|--------------------------------------|---------------------------------------|
| Interface uniformity        | No — dual calling convention         | ABI-dependent: WIT or binary port-map |
| Security isolation          | Inconsistent (weakest codec wins)    | Uniform sandbox per codec             |
| Portability                 | Limited by native codecs             | Platform-agnostic `.wasm` artifact    |
| Step boundary copies        | 0–2; varies by step backend pair     | 1 (Core→Core); 2 (Component Model)    |
| Static composition          | Not applicable                       | Available (Component Model, fixed topology) |
| Distribution uniformity     | No — packages + `.wasm` files        | Yes — `.wasm` artifacts only          |
| Migration barrier           | Low — existing codecs reusable       | Higher — requires Wasm toolchain      |
| Hardware access             | Full (native steps)                  | Limited to WASI + host functions      |
| Debugging                   | Native steps: standard tools; Wasm steps: runtime-aware | Runtime-aware throughout |
| Orchestrator code paths     | Multiple (by codec type)             | One                                   |

## Part 2: Core Module ABI vs. Component Model

This section covers the interface style used for the Wasm codecs — how data
crosses the host–codec boundary. This choice is orthogonal to Part 1 but
interacts with it (discussed in Part 3).

### Definitions

**Core module ABI** (hereafter "Core ABI"): The codec is compiled as a Core
WebAssembly module. Its exported functions use only Wasm's numeric primitives
(`i32`, `i64`, `f32`, `f64`). Data exchange uses manually agreed
pointer/length conventions; the host reads and writes the module's linear
memory directly using `Memory.write()` and `Memory.read()`.

**Component Model**: The codec is compiled as a Component Model component, a
distinct binary format (different magic bytes: `0d 00 01 00` vs. `01 00 00 00`
for Core modules). Its interface is declared in WIT; the Canonical ABI handles
all type marshalling automatically. The component's linear memory is hidden
from the host — there is no `Memory.write()` path available.

### Core Module ABI

**Pros**

- **Fast host↔codec data transfer in all host languages, including Python.**
  The host exchanges data with the module via `Memory.write()` and
  `Memory.read()`, which are single C-level calls regardless of host language.
  No per-byte iteration.

- **Broadest runtime compatibility.** Every Wasm runtime supports Core modules.
  Component Model support is a superset that not all runtimes implement.

- **Simpler binary format.** Core modules have less format overhead. Inspection
  with tools like `wasm2wat` is more direct.

- **Minimal toolchain requirements.** Any language that can target
  `wasm32-wasi` produces a Core module without needing Component Model toolchain
  support.

**Cons**

- **No machine-readable interface.** There is no formal description of what the
  module expects or produces. Violations surface as silent data corruption, not
  load-time errors.

- **No automated code generation.** Every host language must independently
  implement the marshalling protocol.

- **No load-time interface verification.** The runtime can validate function
  type signatures but cannot verify that the codec interprets arguments
  correctly.

- **No ecosystem composition.** Core modules cannot be statically composed.

- **Manual memory management.** The host must manage allocations in the module's
  linear memory.

- **No rich type system at the boundary.** All values must be encoded as
  integers or floats. Complex types require a manual serialization convention.

### Component Model

**Pros**

- **Interface verification at load time.** The runtime checks that the
  component's WIT exports match the expected interface before any code runs.

- **Automated code generation.** WIT toolchains generate all marshalling glue.
  Codec authors work with native types in their language.

- **Language-agnostic interoperability.** Any language with a Component Model
  toolchain can implement the same WIT interface through one uniform code path.

- **Rich type system.** WIT supports strings, `list<T>`, `tuple<T, U>`,
  `option<T>`, `result<T, E>`, `record`, `variant`, and `resource`.

- **Memory isolation by design.** The component's linear memory is not
  accessible to the host outside the WIT interface.

- **Self-describing binary.** Components embed their WIT interface declarations
  in the binary.

- **Static composition.** `wasm-tools compose` can merge a fixed pipeline
  topology into one component, eliminating inter-step copies.

- **Ecosystem alignment.** The Wasm ecosystem — WASI P2, warg/wa.dev,
  Bytecode Alliance tooling — is converging on the Component Model.

**Cons**

- **Canonical ABI cost under Python hosts.** wasmtime-py iterates each byte
  individually in Python. Measured throughput: ~1.7 MB/s. A native orchestrator
  eliminates this discrepancy.

- **Linear memory hidden from host.** There is no `Memory.write()` shortcut
  from a Python host. The Canonical ABI is the only data exchange path.

- **Narrower runtime support.** Requires Wasmtime or Wasmer 4+.

- **WIT versioning coordination.** A breaking WIT change requires updating every
  codec and every consumer.

- **Toolchain maturity varies by language.** Rust and Python have mature support.
  Go and Zig have partial or evolving support.

- **Binary size overhead.** Component wrappers add format overhead (typically
  kilobytes).

### Hybrid: Both Core ABI and Component Model in One Pipeline

**Pros**

- **Per-codec optimization.** High-throughput codecs can use Core ABI for fast
  transfers under a Python orchestrator. Complex-interface codecs can use
  Component Model for verification and code generation.

- **Migration path.** Existing Core module codecs can coexist with newer
  Component Model codecs during a transition.

**Cons**

- **Dual calling convention in the orchestrator.** Two distinct codec invocation
  code paths that cannot be unified.

- **No uniform interface contract.** Core module steps have no load-time
  verification.

- **No static composition across the whole pipeline.** Cannot link Core modules
  to Component Model components.

- **Security asymmetry.** Core module linear memory is accessible to the host;
  component memory is not.

- **Not a stable architectural target.** The hybrid state introduces complexity
  that pays off only during a migration.

### Summary: Core ABI vs. Component Model vs. Hybrid

| Property                        | Core Module ABI                       | Component Model                    | Hybrid                             |
|---------------------------------|---------------------------------------|------------------------------------|-------------------------------------|
| Interface verification at load  | No                                    | Yes (WIT)                          | Partial (Component steps only)     |
| Automated code generation       | No                                    | Yes (wit-bindgen et al.)           | Partial                             |
| Data transfer speed (Python)    | ~ctypes speed (~10 GB/s)              | 1.4–1.9 MB/s (Python binding)      | Mixed                               |
| Data transfer speed (native)    | ~10 GB/s                              | ~10 GB/s                           | ~10 GB/s                            |
| Linear memory accessible        | Yes (host uses Memory.read/write)     | No (hidden by design)              | Mixed                               |
| Static composition              | No                                    | Yes (fixed topology only)          | No (cannot span both types)         |
| Runtime support breadth         | All runtimes                          | Wasmtime, Wasmer 4+                | All runtimes (Core steps only)      |
| Rich types at boundary          | No (integers/floats only)             | Yes (WIT type system)              | Partial                             |
| Self-describing binary          | No                                    | Yes                                | Partial                             |
| Memory management               | Manual (host manages alloc/free)      | Automated (Canonical ABI)          | Mixed                               |
| Security model                  | Sandboxed; memory readable by host    | Full isolation                     | Inconsistent per codec              |
| Ecosystem alignment             | General Wasm                          | Bytecode Alliance / WASI P2        | Mixed                               |
| Orchestrator code paths         | One                                   | One                                | Two                                 |

## Part 3: Interaction of the Two Axes

The two axes are independent choices but they interact in practice.

**A mixed pipeline still requires a Core-or-Component decision for its Wasm
subset.** Allowing native codecs does not resolve the Core ABI vs. Component
Model question for the Wasm steps. Both choices must be made.

**The Python canonical ABI performance problem applies only to Component
Model + Python host.** Using Core ABI with a Python orchestrator avoids the
~1.7 MB/s bottleneck entirely. If the orchestrator is native (Rust, C, Go), the
bottleneck disappears and both Core ABI and Component Model achieve memcpy speed.
The correct fix for a Python orchestrator that uses the Component Model is a
native orchestrator — not a switch to Core ABI.

**Static composition is only available for all-Wasm + Component Model +
fixed topology.** It requires all pipeline steps to be Component Model components
and the topology to be known at build time. Mixed pipelines, Core ABI pipelines,
and dynamic DAG pipelines cannot use it.

**The long-term ecosystem trajectory favors Component Model.** WASI P2, warg/wa.dev,
and Bytecode Alliance tooling are converging on the Component Model as the
standard Wasm interface layer.

**Static composition and native orchestration have different cost profiles.**
Static composition eliminates inter-step copies by merging the pipeline at build
time. A native orchestrator eliminates the Python binding cost but keeps the
per-step copies. Both improvements are independent and composable.

## Part 4: Cylf's Position

Cylf supports both native and Wasm codecs. The standard library provides native
compiled implementations of widely-used codecs for maximum performance. The WASM
runtime provides extensibility through portable, sandboxed modules.

This is a pragmatic choice. An all-Wasm architecture would be cleaner (uniform
interface, uniform security, uniform distribution), but native codecs exist for
performance-critical paths and for codecs where hardware access beyond WASI is
required. The standard library serves as the trusted, high-performance core;
WASM serves as the extensibility mechanism for everything else.

For the Wasm codecs, Cylf supports both Core module ABI and Component Model.
The Component Model is the preferred path — it provides load-time interface
verification, automated code generation, and ecosystem alignment. Core module
ABI is available as an alternative when needed. A native orchestrator
(Rust or C) eliminates the Canonical ABI performance difference between the two,
making the choice primarily about toolchain and interface richness rather than
performance.

The proof-of-concept implementation (chonkle) demonstrated all three backends
(Component Model, Core Wasm, native) in a Python host, which informed the
performance characterization in this document. The reference runtime will be a
native host, where the Canonical ABI bottleneck does not apply.

## Appendix: Benchmark Data

Performance figures cited in this document are from measurements taken in the
proof-of-concept implementation using wasmtime-py 41 and wasmtime-rs 41 on the
same `.wasm` binary and same buffer sizes.

| Measurement                                      | Value          |
|--------------------------------------------------|----------------|
| wasmtime-py Canonical ABI throughput (`list<u8>`) | 1.4–1.9 MB/s  |
| wasmtime-py raw bench (2 MB, 3 iterations)        | ~1.7 MB/s     |
| wasmtime-rs typed bindings (2 MB, 3 iterations)   | ~9,100 MB/s   |
| Per-step copies (Component Model, any host)       | 2             |
| Per-step copies (Core→Core, any host)             | 1             |
| Per-step copies after static composition (Component Model, fixed topology) | 2 total |

See [Performance & Data Copy Accounting](02_perf_accounting.md) for full benchmark
methodology and per-codec measurements.
