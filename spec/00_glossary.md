# Glossary

Quick reference for terms used throughout the Cylf specification.

**Awareness.** A classification of the minimum knowledge a codec needs about the
data it transforms, beyond raw bytes. The four levels — pure-bytes,
stride-aware, type-aware, and structure-aware — determine what parameters a
format driver must extract from format metadata and supply to the codec. See
[Awareness Taxonomy](03_codecs/01_signature.md#awareness-taxonomy).

**Canonical ABI.** The type marshalling layer defined by the WebAssembly
Component Model. It handles serialization and deserialization of typed values
(including `list<u8>` byte buffers) across the host–component boundary. Each
crossing incurs a data copy.

**Codec.** A stateless, typically bidirectional data transformation. A codec's
interface is defined by its signature. Codecs may be implemented as native
libraries, Core Wasm modules, or Component Model Wasm components. A pipeline is
also a codec.

**Codec inventory.** The reference catalog of known codecs — their signatures,
format aliases, awareness classifications, and pipeline decompositions. It is a
specification resource, not a runtime component. See
[Codec Inventory](03_codecs/03_inventory.md).

**Component Model.** A WebAssembly standard that adds typed interfaces (WIT),
automatic code generation, memory isolation, and static composition on top of
Core Wasm. Component binaries have distinct magic bytes (`0d 00 01 00`). See
[Codec Contract](04_runtime/01_codec_contract.md#component-model-wasm).

**Core Wasm.** A WebAssembly module using only Wasm's numeric primitives for its
exported functions. Data exchange uses a binary port-map format in linear memory.
Core Wasm binaries have magic bytes `01 00 00 00`. See
[Codec Contract](04_runtime/01_codec_contract.md#core-wasm).

**Format driver.** A module responsible for a specific data format (Zarr, TIFF,
Parquet, HDF5, etc.). Format drivers read format-specific metadata, construct
pipeline definitions describing the codec steps for a chunk, and hand them to
the pipeline engine for execution. Cylf defines the interface format drivers
target but does not define the drivers themselves.

**Pipeline.** A codec whose internal logic is expressed as a directed acyclic
graph (DAG) of codec steps. A pipeline declares its interface using the same
`encode`/`decode` structure as a leaf codec, and additionally defines steps and
wiring that implement each direction. A pipeline is a codec. See
[Pipeline Model](03_codecs/02_pipeline.md).

**Pipeline engine.** The runtime component that receives a pipeline definition,
resolves each step to a codec implementation, validates signatures against the
pipeline's requirements, and executes the DAG.

**Port.** A named, typed channel on a codec's interface. Ports typed `bytes`
carry data being transformed; ports with scalar types (`int`, `string`, `bool`,
`uint[]`, `dtype_desc`) carry parameters that configure codec behavior.

**Port map.** The wire format used to pass data through a codec's ports at
runtime: an ordered list of `(name, bytes)` pairs, where each entry corresponds
to a port on the codec's interface. Non-bytes values (parameters) are serialized
as UTF-8 JSON strings within port-map entries.

**Signature.** A JSON object that declares a codec's ports — the named, typed
inputs it accepts and outputs it produces — in each direction (`encode` and
`decode`). Every codec carries its own signature, which the pipeline engine
validates against pipeline requirements before execution. See
[Signature Model](03_codecs/01_signature.md).

**Standard library.** The set of native compiled codec implementations bundled
with the Cylf runtime, covering widely-used codecs (zstd, lz4, deflate,
shuffle, delta, etc.) for maximum performance.

**Step.** A single codec invocation within a pipeline. Each step references a
`codec_id` and maps its input ports to sources via wiring references.

**Wiring reference.** A dot-notation string identifying the source of a step's
input value within a pipeline: `inputs.<port>` for pipeline-level inputs,
`constants.<name>` for pipeline-level constants, or
`steps.<step_name>.<port>` for the output of another step.
