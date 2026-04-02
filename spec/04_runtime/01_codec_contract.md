# Codec Contract

This document defines the contract between the Cylf pipeline engine and codec
implementations. All codecs present the same logical interface regardless of
backend. Cylf supports three codec backends: Component Model Wasm, Core Wasm,
and Native.

## Interface

Every codec exports three operations:

- **`signature`** — returns the codec's signature (a JSON object declaring its
  interface).
- **`encode`** — accepts named inputs and produces named outputs for the encode
  direction.
- **`decode`** — accepts named inputs and produces named outputs for the decode
  direction.

Unidirectional codecs export all three operations but return an error from the
unsupported direction.

`encode` and `decode` exchange data through **port maps**: ordered lists of
`(name, bytes)` pairs. Port names route data between pipeline steps — the
engine matches step outputs to downstream step inputs by name. Codec parameters
(non-bytes inputs like compression level or element size) arrive as named port
entries serialized as UTF-8 JSON bytes.

### Signature

Every codec must carry a signature: a JSON object declaring the codec's
interface in each direction it supports. The `signature` export must return
that JSON signature object when called. For Wasm codecs, the signature is also
embedded in the `.wasm` binary as a `cylf:signature` custom section. See the
[Signature Model](../03_codecs/01_signature.md) for the full specification of
the signature format, type vocabulary, and port descriptor fields.

#### Validation

The pipeline engine validates signatures during pipeline preparation by
verifying:

- each step's input wiring satisfies its codec's signature in the active
  direction. Every required input (those that are not explicitly
  `"required": false`) must be wired.
- any input wiring referencing a preceding step's output names a port declared
  in that step's outputs for the active direction.
- all connected ports are type-compatible.

## Backends

### Component Model Wasm

Component Model codecs are `.wasm` components (magic bytes `0d 00 01 00`) that
export a typed WIT interface. The Canonical ABI handles all type marshalling
automatically. Codec authors work with high-level types in their language of
choice; the toolchain generates memory management and serialization glue.

This is the recommended backend for new codecs. It provides load-time interface
verification, automated code generation, language-agnostic authoring, and
ecosystem alignment with WASI P2 and the Bytecode Alliance toolchain.

A Component Model codec must export the `cylf:codec/transform` interface.
The WIT definition:

```wit
package cylf:codec@0.1.0;

interface transform {
    type port-name = string;
    type port-map = list<tuple<port-name, list<u8>>>;

    signature: func() -> list<u8>;
    encode: func(inputs: port-map) -> result<port-map, string>;
    decode: func(inputs: port-map) -> result<port-map, string>;
}

world codec {
    export transform;
}
```

`signature` returns the codec's signature as UTF-8 JSON bytes.

#### Requirements

- The component must export the `transform` interface under the key
  `"cylf:codec/transform"`.
- The component must be WASIp2-compatible.
- Return `Ok(port-map)` on success. Return `Err(string)` on failure; the engine
  surfaces the error string.
- A unidirectional codec must still export both functions (the WIT interface
  requires it). The unsupported direction should return
  `Err("encode not supported")` or `Err("decode not supported")`.

### Core Wasm

Core Wasm codecs are `.wasm` modules (magic bytes `01 00 00 00`) that export
raw functions using Wasm's numeric primitives (`i32`, `i64`, `f32`, `f64`).
Data exchange uses a binary port-map wire format; the host reads and writes
the module's linear memory directly.

Core Wasm provides faster host-to-module data transfer under Python hosts (~10
GB/s via `Memory.read()`/`Memory.write()` vs ~1.7 MB/s for the Component Model
Canonical ABI under wasmtime-py). Under a native (Rust/C) host, both backends
achieve memcpy speed and the performance difference disappears. See
[Performance & Data Copy Accounting](../05_design/02_perf_accounting.md) for
measurements.

A Core Wasm codec is a `wasm32-wasi` reactor module that exports memory,
allocation functions, and `signature`/`encode`/`decode` entry points.

#### Required exports

| Export | Signature | Description |
|---|---|---|
| `memory` | Memory | Linear memory (provided by the Wasm toolchain) |
| `alloc` | `(size: i32) -> i32` | Allocate `size` bytes; return pointer or 0 on failure |
| `dealloc` | `(ptr: i32, size: i32)` | Free a buffer |
| `signature` | `() -> i64` | Return signature JSON; packed `(ptr << 32) \| len` |
| `encode` | `(port_map_ptr: i32, port_map_len: i32) -> i64` | Encode operation |
| `decode` | `(port_map_ptr: i32, port_map_len: i32) -> i64` | Decode operation |

#### Port-map binary format

All multi-byte integers are little-endian.

```
u32: entry_count
For each entry:
  u32: name_len
  u8[name_len]: name (UTF-8, not null-terminated)
  u32: data_len
  u8[data_len]: data
```

No padding or alignment between fields.

#### Calling convention

1. The host calls `alloc(input_size)` to reserve space in the module's linear
   memory.
2. The host writes the serialized port-map into the allocated region via
   `Memory.write()`.
3. The host calls `encode(ptr, len)` or `decode(ptr, len)`. The codec takes
   ownership of the input buffer and is responsible for freeing it.
4. The codec returns a packed `i64`: `(output_ptr << 32) | output_len`. Return
   `0` to signal an error.
5. The host reads the output port-map from linear memory via `Memory.read()`.
6. The host calls `dealloc(output_ptr, output_len)` to free the output buffer.

The memory ownership semantics for the buffer returned by `signature` are not
yet defined — see [Open Questions](../06_future.md#core-wasm-signature-memory-ownership).

### Native

Native codecs run directly in the host process as compiled libraries. No
sandbox, no Wasm runtime overhead. Zero-copy data transfer between adjacent
native codecs. Native codecs have unrestricted access to hardware (SIMD, GPU,
hardware crypto).

The standard library of widely-used codecs (zstd, lz4, deflate, shuffle,
delta, etc.) consists of native implementations.

Native codecs are compiled libraries (shared or statically linked) that expose
a C ABI.

#### Required exports

Native codecs must export the following C-callable symbols:

| Export | Signature | Description |
|---|---|---|
| `signature` | `(out_len: *mut u32) -> *const u8` | Return a pointer to the signature JSON and write its length to `out_len` |
| `encode` | `(port_map_ptr: *const u8, port_map_len: u32, out_len: *mut u32) -> *const u8` | Encode operation |
| `decode` | `(port_map_ptr: *const u8, port_map_len: u32, out_len: *mut u32) -> *const u8` | Decode operation |
| `dealloc` | `(ptr: *const u8, len: u32)` | Free a buffer returned by `encode` or `decode` |

`encode` and `decode` accept a serialized port-map and return a pointer to the
output port-map, writing its length to `out_len`. Return `NULL` to signal an
error. The port-map uses the same [binary format](#port-map-binary-format)
defined in the Core Wasm section.

The `signature` function returns a pointer to the codec's signature as UTF-8
JSON bytes. The returned buffer must remain valid for the lifetime of the
loaded library (typically a static allocation).

## Backend Detection

The pipeline engine detects the backend from the resolved artifact:

- `.wasm` file with component version header (`0d 00 01 00`) → Component Model
- `.wasm` file with core version header (`01 00 00 00`) → Core Wasm
- no `.wasm` file (resolved to a native implementation) → Native

The pipeline definition is agnostic to backend. The same pipeline executes
correctly regardless of which backends its constituent codecs use.

## Further Reading

- [Signature Model](../03_codecs/01_signature.md): full specification of the
  codec signature format
- [Pipeline Model](../03_codecs/02_pipeline.md): how codecs compose into
  executable DAGs
- [Performance & Data Copy Accounting](../05_design/02_perf_accounting.md):
  copy counts and throughput across backends
- [Pipeline Architecture Tradeoffs](../05_design/01_pipeline_arch.md): design
  rationale for supporting multiple backends
