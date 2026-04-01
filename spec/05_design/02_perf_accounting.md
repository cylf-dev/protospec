# Performance & Data Copy Accounting

This document covers the performance characteristics of the Wasm codec path:
how many data copies occur at each step boundary, why they occur, what
determines their cost, and what approaches have been explored to reduce them.
These findings informed the Cylf architecture — in particular, the decision to
support both native and Wasm codecs, and the requirement for a native (Rust or
C) orchestrator for production use.

The measurements in this document were taken in the proof-of-concept
implementation (chonkle) using wasmtime-py 41 and wasmtime-rs 41.

---

## Fundamental Constraint

Wasm's sandbox model means a module can only read and write its own linear
memory — it cannot dereference a host pointer or access any other module's
memory. The host can read and write the module's linear memory, but not the
reverse. This isolation is why data copies at step boundaries are unavoidable:
data must be copied in before processing and copied out after.

---

## Per-Invocation Copy Anatomy

Each Wasm codec invocation requires at least one copy in each direction
regardless of host language:

```text
     host bytes
         │
      copy in            (lower — host → Wasm linear memory)
         │
         ▼
Wasm linear memory
         │
    codec runs
         │
         ▼
Wasm linear memory
         │
      copy out           (lift — Wasm linear memory → host)
         │
         ▼
     host bytes
```

The *cost* of these copies depends on the Wasm interface style and the host
language.

---

## Copy Counts by Backend Combination

The table below shows the number of copies per inter-step edge for different
combinations of adjacent codec backends.

| Source → Destination | Copies | Mechanism |
| --- | --- | --- |
| Native → Native | 0 | In-process reference; no copy |
| Native → Core Wasm | 1 | Host writes into module's linear memory |
| Core Wasm → Native | 1 | Host reads from module's linear memory |
| Core Wasm → Core Wasm | 1 | Transfer between linear memories (`memcpy`) |
| Any → Component Model | 2 | Canonical ABI lift + lower |
| Component Model → Any | 2 | Canonical ABI lift + lower |

Component Model edges always incur 2 copies because the component's linear
memory is hidden from the host — data must be lifted out through the Canonical
ABI, then lowered back in at the destination. Core Wasm edges can achieve 1 copy
because the host has direct access to both modules' linear memories.

---

## The Canonical ABI Bottleneck Under Python

The 2 copies per Component Model edge are architecturally unavoidable. However,
their *cost* varies dramatically by host language.

### Measurement: ABI throughput

Timing instrumentation around codec calls showed that `fn()` time scales
linearly with total bytes through the ABI at approximately 1.4–1.9 MB/s under
the Python host:

| Codec | ABI traffic | fn() time | Throughput |
|---|---|---|---|
| identity (memcpy only) | 2.00 MB | 1.320s | 1.52 MB/s |
| predictor2 (real work) | 2.00 MB | 1.295s | 1.54 MB/s |
| identity | 4.00 MB | 2.771s | 1.44 MB/s |
| predictor2 | 4.00 MB | 2.507s | 1.60 MB/s |

The identity codec performs only a `malloc` + `memcpy` inside Wasm. Its times
are nearly identical to predictor2 at equal ABI traffic. The cost is ABI
lifting/lowering, not codec computation.

### Measurement: lowering vs lifting

Two calls with equal total ABI traffic (~2 MB) but opposite data distributions:

| Case | In | Out | fn() time | Throughput |
|---|---|---|---|---|
| encode (mostly lowering) | 2.00 MB | 2 KB | 1.197s | 1.67 MB/s |
| decode (mostly lifting) | 2 KB | 2.00 MB | 1.024s | 1.96 MB/s |

The direction of data flow does not materially affect throughput. Both
directions are bottlenecked equally.

### Root cause: Python binding layer

To isolate whether the bottleneck is in the Canonical ABI itself or in the
Python binding, two minimal benchmarks called the same identity codec with
the same 2 MB buffer — one from Rust, one from Python:

| Host | ABI traffic | fn() time | Throughput |
|---|---|---|---|
| Python (wasmtime-py 41) | 4 MB | ~2.3s | 1.7 MB/s |
| Rust (wasmtime-rs 41) | 4 MB | ~0.001s | 9,113 MB/s |

Rust throughput is approximately 5,000× faster at the same Wasm binary and
buffer size.

The root cause is in wasmtime-py's type adaptation layer. For `list<u8>`, the
Python binding iterates every byte individually:

```python
for e in val:                                         # O(N) Python loop
    element.convert_to_c(store, e, pointer(raw.data[i]))
```

A 2 MB buffer triggers ~2 million Python loop iterations, ~2 million
`isinstance()` checks, and ~2 million ctypes field writes per copy direction.
There is no bulk `memcpy` path.

The Rust binding's `Lower` trait for `list<u8>` writes the byte slice into Wasm
linear memory via a single `memory.write()` — one `memcpy`.

### Why the Python host cannot bypass the Canonical ABI

The Component Model hides its linear memory from the host:

```python
mem_idx = instance.get_export_index(store, "memory")
# → None. Memory is not accessible from a component instance.
```

There is no `Memory.write()` shortcut available from a Component Model
component instance. The Canonical ABI is the only data exchange path.

Core Wasm modules do not have this limitation — their linear memory is
directly accessible, which is why Core Wasm transfers run at `memcpy` speed
even from Python.

---

## Architectural Implications

### A native orchestrator is required for production

The Canonical ABI bottleneck is entirely in the Python binding layer, not in
the Component Model architecture. A native (Rust or C) orchestrator eliminates
the bottleneck: both copy directions run at `memcpy` speed (~10 GB/s). At that
point, the 2 copies per Component Model edge complete in under 1 ms for typical
chunk sizes (sub-megabyte to low single-digit megabytes), making the copy
overhead negligible relative to codec computation.

The Python orchestrator should be treated as a development and test host, not
a production runtime.

### The native standard library avoids copies entirely for hot paths

When two adjacent pipeline steps are both native codecs (from the standard
library), data passes as an in-process reference — zero copies. For
performance-critical paths where every codec in the pipeline has a native
implementation, this eliminates copy overhead entirely. This is one motivation
for Cylf's standard library of native codecs alongside the Wasm runtime.

### Component Model vs Core ABI is a toolchain question under a native host

Under a native orchestrator, both Core Wasm and Component Model achieve
`memcpy` throughput. The performance difference that motivated Core Wasm support
in the Python host disappears. The choice between the two becomes about
interface richness (Component Model provides load-time verification, code
generation, and a rich type system) and runtime compatibility (Core Wasm runs
on every runtime; Component Model requires Wasmtime or Wasmer 4+).

---

## Copy Reduction Approaches

Several approaches to reducing copy counts have been explored.

### Shared memory between Core modules

An earlier design used a single shared `wasmtime.Memory` for all Core Wasm
modules, enabling zero-copy transfers between steps (pointer hand-off rather
than `memcpy`). This was abandoned because every module's data segments target
the same low addresses in linear memory — when a second module is instantiated,
it overwrites the first module's data segments, preventing simultaneous
instantiation.

### Multi-memory

The Wasm multi-memory feature (part of Wasm 3.0) could in theory solve the
shared-memory problem: each module keeps a private memory for data segments and
heap, and all modules share a second memory for the data plane. In practice,
C and Rust compilers (LLVM) cannot emit load/store instructions targeting a
non-zero memory index. Library-wrapping codecs (zlib, zstd, etc.) would need to
copy data between shared and private memory on every call — worse than the
current design.

### Static pipeline composition

`wasm-tools compose` can merge multiple Component Model components into a single
component. Intra-component calls share linear memory, eliminating inter-step
copies. A composed N-step pipeline pays only 2 copies total (one in, one out)
regardless of step count.

This requires the pipeline topology to be known at build time, which is
incompatible with dynamic DAG pipelines assembled at runtime. It also only works
for Component Model components.

### Component Model resources

Component Model resources are handle-based types that avoid copying data when
passing references between components. Two patterns exist for sharing resources:

**Pattern 1 (direct composition)** achieves 1 copy per edge but requires static
compile-time coupling between components — each component must hardcode its
neighbor's package in its WIT. This is incompatible with pipelines assembled at
runtime.

**Pattern 2 (shared buffer-store)** uses a third component as an intermediary.
Both the producer and consumer import from the same buffer-store instance.
However, data still crosses two component boundaries per edge (producer →
buffer-store, buffer-store → consumer), giving the same 2 copies per edge as
value passing with `list<u8>`.

### Caller-supplied buffers (not yet available)

The WASI roadmap includes caller-supplied buffers, which would allow a host to
pass a pre-allocated buffer for the module to write into, eliminating the
copy-out step. This is not yet part of the spec.

### Native Python extension for Canonical ABI

A Rust extension (PyO3 + wasmtime-rs) could replace wasmtime-py's per-byte
Python iteration with wasmtime-rs's bulk `memory.write()` calls. This would
bring Component Model copies to `memcpy` speed (~10 GB/s) without changing the
copy count. This is a mitigation for the Python host specifically; a native
orchestrator achieves the same result without a separate extension.

---

## Summary

| Property | Value |
|---|---|
| Copies per edge: Native → Native | 0 |
| Copies per edge: Core → Core | 1 |
| Copies per edge: Component Model | 2 |
| Copies after static composition (Component Model) | 2 total |
| Copy throughput: Python host, Component Model | ~1.7 MB/s |
| Copy throughput: Python host, Core Wasm | ~10 GB/s |
| Copy throughput: native host, any Wasm backend | ~10 GB/s |
| Root cause of Python bottleneck | Per-byte iteration in wasmtime-py binding |
| Fix | Native (Rust/C) orchestrator |

The copy counts are architectural — they follow from Wasm's sandbox model and
cannot be eliminated without changes to the Wasm spec (caller-supplied buffers)
or static pipeline composition. The copy *cost* under Python is a binding
implementation issue, fully resolved by a native orchestrator.