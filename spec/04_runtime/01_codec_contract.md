# Codec Contract

This document specifies the interface contract between the Cylf pipeline engine
and codec implementations. All codecs — regardless of backend — present the
same logical interface: an `encode` operation and a `decode` operation (or just
one, for unidirectional codecs), each accepting named inputs and producing named
outputs as defined in the codec's signature.

Cylf supports three codec backends. The logical interface is the same across
all three; the backends differ in how data crosses the module boundary.

---

## Backends

### Component Model Wasm

Component Model codecs are `.wasm` components (magic bytes `0d 00 01 00`) that
export a typed WIT interface. The Canonical ABI handles all type marshalling
automatically. Codec authors work with high-level types in their language of
choice; the toolchain generates memory management and serialization glue.

This is the recommended backend for new codecs. It provides load-time interface
verification, automated code generation, language-agnostic authoring, and
ecosystem alignment with WASI P2 and the Bytecode Alliance toolchain.

### Core Wasm

Core Wasm codecs are `.wasm` modules (magic bytes `01 00 00 00`) that export
raw functions using Wasm's numeric primitives (`i32`, `i64`, `f32`, `f64`).
Data exchange uses a binary port-map wire format; the host reads and writes
the module's linear memory directly.

Core Wasm provides faster host-to-module data transfer under Python hosts
(~10 GB/s via `Memory.read()`/`Memory.write()` vs ~1.7 MB/s for the Component
Model Canonical ABI under wasmtime-py). Under a native (Rust/C) host, both
backends achieve memcpy speed and the performance difference disappears. See
[Performance & Data Copy Accounting](../05_design/02_perf_accounting.md) for measurements.

### Native

Native codecs run directly in the host process as compiled libraries. No
sandbox, no Wasm runtime overhead. Zero-copy data transfer between adjacent
native codecs. Native codecs have unrestricted access to hardware (SIMD, GPU,
hardware crypto).

The standard library of widely-used codecs (zstd, lz4, deflate, shuffle,
delta, etc.) consists of native implementations.

---

## Signature

Every codec must carry a signature: a JSON object declaring the codec's
interface in each direction it supports. For Wasm codecs, the signature is
embedded in the `.wasm` binary as a custom section. For native codecs, the
signature is bundled alongside the implementation.

### Format

```json
{
  "codec_id": "zstd",
  "encode": {
    "inputs": {
      "bytes": {"type": "bytes"},
      "level": {"type": "int", "default": 3},
      "dictionary": {"type": "bytes", "required": false}
    },
    "outputs": {
      "bytes": {"type": "bytes"}
    }
  },
  "decode": {
    "inputs": {
      "bytes": {"type": "bytes"},
      "dictionary": {"type": "bytes", "required": false}
    },
    "outputs": {
      "bytes": {"type": "bytes"}
    }
  }
}
```

### Fields

- **`codec_id`** (string, required) — canonical codec identifier (e.g.
  `"zstd"`, `"tiff-predictor-2"`).
- **`encode`** (object, optional) — the codec's interface when encoding. Must
  contain `inputs` and `outputs`.
- **`decode`** (object, optional) — the codec's interface when decoding. Must
  contain `inputs` and `outputs`.

At least one of `encode` or `decode` must be present. A codec that defines only
one is unidirectional.

Each direction's `inputs` is a map of port descriptors. Each port descriptor
has:

- `type` (string, required) — one of: `"bytes"`, `"string"`, `"int"`, `"uint"`,
  `"float"`, `"bool"`, `"uint[]"`, `"dtype_desc"`.
- `required` (bool, optional, default `true`) — whether the caller must supply
  this input. A `default` value implies `required: false`.
- `default` (optional) — default value when not supplied.

Each direction's `outputs` is a map of port descriptors, each with only `type`.

See the [Signature Model](../03_codecs/01_signature.md) for the full specification.

### Validation

The pipeline engine validates signatures during pipeline preparation:

- Each step's input wiring is checked against the codec's signature for the
  active direction. Every required input (those without a `default`) must be
  wired.
- Output port references from downstream steps are checked against the codec's
  declared outputs for the active direction.
- Type compatibility between connected ports is verified.

---

## Component Model Interface

A Component Model codec must export the `chonkle:codec/transform` interface.
The WIT definition:

```wit
package chonkle:codec@0.1.0;

interface transform {
    type port-name = string;
    type port-map = list<tuple<port-name, list<u8>>>;

    encode: func(inputs: port-map) -> result<port-map, string>;
    decode: func(inputs: port-map) -> result<port-map, string>;
}

world codec {
    export transform;
}
```

Port maps are ordered lists of `(port-name, bytes)` pairs. Port names are
conventions used to route data between pipeline steps — the engine matches
step outputs to downstream step inputs by name.

Codec parameters (non-bytes inputs like compression level or element size)
arrive as named port entries serialized as UTF-8 JSON bytes.

### Requirements

- The component must export the `transform` interface under the key
  `"chonkle:codec/transform"`.
- The component must be WASIp2-compatible.
- Return `Ok(port-map)` on success. Return `Err(string)` on failure; the engine
  surfaces the error string.
- A unidirectional codec must still export both functions (the WIT interface
  requires it). The unsupported direction should return
  `Err("encode not supported")` or `Err("decode not supported")`.

### Signature embedding

The codec's signature must be embedded in the `.wasm` binary as a custom
section at build time.

---

## Core Wasm Interface

A Core Wasm codec is a `wasm32-wasi` reactor module that exports memory,
allocation functions, and encode/decode entry points.

### Required exports

| Export | Signature | Description |
|---|---|---|
| `memory` | Memory | Linear memory (provided by the Wasm toolchain) |
| `alloc` | `(size: i32) -> i32` | Allocate `size` bytes; return pointer or 0 on failure |
| `dealloc` | `(ptr: i32, size: i32)` | Free a buffer |
| `encode` | `(port_map_ptr: i32, port_map_len: i32) -> i64` | Encode operation |
| `decode` | `(port_map_ptr: i32, port_map_len: i32) -> i64` | Decode operation |

### Port-map binary format

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

### Calling convention

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

### Signature embedding

Same as Component Model — the signature is embedded as a custom section in the
`.wasm` binary.

---

## Native Interface

Native codecs wrap compiled libraries (e.g. libzstd, zlib) or provide
standalone implementations in the host language. They run in-process with no
sandbox.

### Calling convention

Native codecs implement the same logical interface as Wasm codecs: an `encode`
and a `decode` operation, each accepting a port-map (list of name/bytes pairs)
and returning a port-map or an error.

The specific API depends on the host language. In a Rust host, a native codec
implements a trait with `encode` and `decode` methods. In a Python host, a
native codec implements a class with `call(direction, port_map)` and
`signature()` methods.

### Signature

Native codec signatures use the same JSON format as Wasm codec signatures. The
signature is bundled alongside the implementation (as a JSON file, an embedded
resource, or a programmatic return value from the codec's API).

---

## Backend Detection

The pipeline engine detects the backend from the resolved artifact:

- `.wasm` file with component version header (`0d 00 01 00`) → Component Model
- `.wasm` file with core version header (`01 00 00 00`) → Core Wasm
- No `.wasm` file (resolved to a native implementation) → Native

The pipeline definition is agnostic to backend. The same pipeline executes
correctly regardless of which backends its constituent codecs use.

---

## Further Reading

- [Signature Model](../03_codecs/01_signature.md) — full specification of the codec
  signature format
- [Pipeline Model](../03_codecs/02_pipeline.md) — how codecs compose into executable DAGs
- [Performance & Data Copy Accounting](../05_design/02_perf_accounting.md) — copy counts and
  throughput across backends
- [Pipeline Architecture Tradeoffs](../05_design/01_pipeline_arch.md) — design rationale
  for supporting multiple backends