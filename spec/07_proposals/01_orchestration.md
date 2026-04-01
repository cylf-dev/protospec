# Proposal: cylf Runtime Plan Format and Execution Model

> [!NOTE]
> This is a draft document written by an LLM. It serves to document a
> human-generated idea, and as a platform for iteration and ideation. Do not
> expect it to all be correct or the assertations to be fully vetted.

## Abstract

This document specifies the plan format, execution model, module interfaces,
error model, and observability model for the cylf runtime. The cylf runtime
orchestrates data encoding and decoding pipelines by composing discrete codec,
format driver, and storage driver modules into executable plans. Modules may be
implemented as native libraries, core ABI WebAssembly modules, or WebAssembly
Component Model modules, with a uniform interface across all three tiers. The
runtime manages data movement between module memory spaces, concurrent
execution of independent work, and resource safety for untrusted modules.

This RFC builds on top of the codec pipeline model defined separately. A codec
pipeline — a DAG of codec operations that transforms chunk bytes — is a
standalone concept usable independently by any tooling that has chunk bytes and
a pipeline definition. The plan described here is a higher-level construct that
adds storage I/O, format-specific operations, concurrency, and orchestration
around codec pipelines.

## Motivation

Data in columnar and chunked array formats (Parquet, Zarr, HDF5, TIFF, ORC,
Arrow IPC, and others) is encoded using pipelines of codecs — compression,
shuffling, delta coding, bit packing, and many more. Today, each format library
implements its own codec integration, resulting in duplicated effort,
inconsistent behavior, and difficulty extending the ecosystem with new codecs
or formats.

The cylf runtime provides a common execution substrate: format drivers describe
*what* operations to perform (as a plan), and the runtime handles *how* to
execute them — managing memory, concurrency, module loading, and error
handling. Codec modules are reusable across formats. Format drivers are
decoupled from codec implementations. Storage drivers abstract over I/O
backends. All three are dynamically loadable and extensible.

The system supports three module tiers to balance performance against safety:

- **Native modules** run in the host process with zero-copy data access,
  suitable for trusted, performance-critical paths.
- **Core ABI WebAssembly modules** run in a sandboxed environment with at most
  one data copy per boundary crossing, suitable for most use cases.
- **Component Model WebAssembly modules** run with full Component Model
  isolation at the cost of two copies per boundary crossing, suitable for
  maximum portability and the broadest module ecosystem.

A single plan executes correctly regardless of which tiers its constituent
modules use. The runtime resolves module tiers and manages data movement
transparently.

## Terminology

**Plan.** A directed acyclic graph (DAG) of operations that describes a
complete data processing task — reading, decoding, and assembling data from a
stored format, or encoding and writing data into a stored format. A plan is the
unit of execution submitted to the runtime.

**Node.** A single operation within a plan. Each node has a type, typed input
and output ports, and parameters. Nodes are connected by edges that define data
flow.

**Edge.** A directed connection from an output port on one node to an input
port on another. Edges define both data dependencies and execution order.

**Module.** An executable unit that implements one or more operations. Modules
are classified by role (codec, format driver, storage driver) and by tier
(native, core ABI, Component Model).

**Codec.** A module that performs a stateless, bidirectional data
transformation. Codecs implement both encode and decode operations. A codec's
interface is defined by its signature: typed input ports, output ports, and
parameters. Codec pipelines — DAGs of codec operations — are defined separately
and may appear as subgraphs within a plan.

**Format driver.** A module responsible for a specific data format. Format
drivers parse format metadata, construct plans, provide callback functions
referenced by plans (such as dictionary construction or structural assembly),
and produce format metadata during encoding.

**Storage driver.** A module that abstracts over a storage backend (local
filesystem, cloud object storage, etc.), providing a uniform read/write
interface to the runtime and format drivers.

**Tier.** The execution model of a module: native (in-process dynamic library),
core ABI (WebAssembly module with runtime-managed linear memory), or Component
Model (WebAssembly component with canonical ABI isolation). All three tiers
implement the same logical interface. The tier determines how data crosses
module boundaries.

**Orchestrator.** The core of the cylf runtime. The orchestrator validates
plans, resolves and loads modules, manages memory and data movement between
module tiers, schedules concurrent execution, and enforces resource limits. The
orchestrator is always native code.

## Plan Format

### Overview

A plan is a serialized DAG of operations. Plans are produced by format drivers,
validated by the orchestrator, and may be serialized for storage, transmission,
or inspection by external tools. The canonical serialization format is
FlatBuffers, chosen for zero-copy deserialization, compact representation, and
cross-language support.

Plans are versioned. The plan format version is encoded in the serialized
header. The orchestrator rejects plans with unsupported versions. Format
version evolution follows semver: minor versions add optional fields without
breaking existing plans; major versions may change the schema incompatibly.

### Node Types

The plan defines a fixed vocabulary of node types. Each node type has specific
semantics understood by the orchestrator. This vocabulary is intentionally
small; complex format-specific logic is expressed through codec pipelines and
driver callbacks, not through new node types.

#### `read`

An I/O node that reads a byte range from storage into the runtime's buffer
space.

| Port | Direction | Type | Description |
|------|-----------|------|-------------|
| `bytes` | output | bytes | The bytes read from storage |

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `source` | string | yes | Storage source identifier |
| `offset` | uint | yes | Byte offset within the source |
| `length` | uint | yes | Number of bytes to read |

The orchestrator dispatches read nodes to the storage driver associated with
`source`. Read nodes are I/O-bound; the orchestrator schedules them to maximize
I/O concurrency.

#### `write`

An I/O node that writes bytes to storage.

| Port | Direction | Type | Description |
|------|-----------|------|-------------|
| `bytes` | input | bytes | The bytes to write |

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `source` | string | yes | Storage target identifier |
| `offset` | uint | no | Byte offset within the target (if applicable) |
| `ordering` | uint | no | Write ordering constraint (lower values written first) |

The `ordering` parameter expresses sequential write constraints required by
some formats (e.g., Parquet requires the footer to be written last). Nodes with
equal or unspecified ordering may be written concurrently.

#### `codec`

A compute node that invokes a codec or codec pipeline. A codec is a pipeline is
a codec: a single codec is a one-step pipeline, and a pipeline is itself a
codec with a derived signature. The `codec` node type handles both uniformly.

| Field | Description |
|-------|-------------|
| `codec_id` | The identifier of the codec or pipeline to invoke |
| `direction` | `encode` or `decode` |
| `input_bindings` | Maps the codec's input port names to data sources (edges from other nodes, or inline constant values) |

Input ports, output ports, and parameters are determined by the codec's
signature, resolved at plan validation time. The `codec_id` may reference a
single codec module, a pipeline defined in the plan's `pipelines` section (see
below), or a codec/pipeline resolvable from an external registry. The
orchestrator validates that the `input_bindings` match the codec's declared
signature.

When many plan nodes share the same codec pipeline (a common case — e.g., all
chunks in an array using the same encoding), the pipeline is defined once in
the plan's `pipelines` section and referenced by `codec_id` from each node.
Per-node variation is expressed through `input_bindings`: the pipeline's input
ports receive different values (byte sources, parameters) for each invocation.

#### `driver_callback`

A compute node that invokes a function exported by the format driver that
produced the plan.

| Field | Description |
|-------|-------------|
| `function_name` | The name of the exported function on the format driver module |

Input and output ports are declared in the node definition within the plan. The
orchestrator validates that the format driver module exports a function with
the given name and a compatible signature.

Driver callbacks are used for format-specific operations that do not fit the
codec model, such as dictionary construction, statistics collection, structural
assembly (e.g., Dremel reassembly), and metadata finalization.

#### `concat`

Combines multiple byte sequences into a single contiguous byte sequence, in
port order.

| Port | Direction | Type | Description |
|------|-----------|------|-------------|
| `chunk_0` .. `chunk_N` | input | bytes | Byte sequences to concatenate |
| `bytes` | output | bytes | The concatenated result |

The number of input ports is declared in the node definition. The output is the
inputs concatenated in port-index order.

#### `slice`

Extracts a contiguous sub-range from a byte sequence. This operation is
logically zero-copy — the runtime may implement it as a view over the source
buffer rather than a physical copy.

| Port | Direction | Type | Description |
|------|-----------|------|-------------|
| `bytes` | input | bytes | Source byte sequence |
| `bytes` | output | bytes | The extracted sub-range |

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `offset` | uint | yes | Byte offset into the input |
| `length` | uint | yes | Number of bytes to extract |

#### `validate`

Verifies data integrity by computing a checksum and comparing it against an
expected value. If validation fails, the node fails with a checksum error.

| Port | Direction | Type | Description |
|------|-----------|------|-------------|
| `bytes` | input | bytes | Data to validate |
| `bytes` | output | bytes | Pass-through of input (unmodified) |

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `algorithm` | string | yes | Checksum algorithm identifier (e.g., `crc32c`, `fletcher32`) |
| `expected` | bytes | yes | Expected checksum value |

#### `choice`

A conditional node that selects one of several branches based on a
discriminator value evaluated at runtime.

| Port | Direction | Type | Description |
|------|-----------|------|-------------|
| (varies) | input | (varies) | Inputs are passed through to the selected branch |
| (varies) | output | (varies) | Outputs come from the selected branch |

| Field | Description |
|-------|-------------|
| `discriminator` | A reference to the input port whose value determines which branch to select |
| `branches` | A map from discriminator values to sub-plan definitions |

All branches must declare the same output port signatures. This constraint
ensures that downstream nodes can be wired without knowledge of which branch
will execute.

#### `scan`

A compute node that reads input data to produce a derived artifact (dictionary,
symbol table, statistics) without producing transformed data output. Used in
encode plans for multi-pass operations.

| Port | Direction | Type | Description |
|------|-----------|------|-------------|
| `bytes` | input | bytes | Source data to scan |
| (varies) | output | (varies) | Derived artifacts (dictionaries, statistics, etc.) |

Scan nodes may be implemented as driver callbacks or as codec invocations,
depending on the operation. The node type exists to signal to the orchestrator
that this is a preparation step whose outputs are metadata, not bulk data —
enabling better scheduling decisions.

### Edges

An edge connects an output port on a source node to an input port on a
destination node.

```
Edge {
    from_node: u32,
    from_port: u16,
    to_node: u32,
    to_port: u16,
}
```

Edges define both data flow and execution ordering. A node may execute only
after all nodes providing its inputs have completed. An output port may be
connected to multiple input ports (fan-out); each consumer receives the same
data. An input port must be connected to exactly one output port.

### Plan Inputs and Outputs

A plan has named external inputs and outputs. External inputs are provided by
the caller when submitting the plan for execution (e.g., source data for an
encode plan). External outputs are the results of plan execution delivered to
the caller (e.g., decoded column arrays from a read plan).

```
PlanInput {
    name: string,
    type: PortType,
    node_id: u32,       // the node whose input port this feeds
    port_index: u16,
}

PlanOutput {
    name: string,
    type: PortType,
    node_id: u32,       // the node whose output port this comes from
    port_index: u16,
}
```

### Parameters

Node parameters are scalar values (int, uint, float, bool) or short strings.
Parameters are encoded in the plan as typed key-value pairs.

```
Parameter {
    name: string,
    type: enum { Int, Uint, Float, Bool, String },
    value: (type-dependent),
}
```

Non-scalar data (dictionaries, symbol tables, large byte sequences) is passed
as data through input ports, not as parameters. This keeps the parameter
encoding simple and pushes complex data into the DAG wiring, which is already
designed to handle it.

### Sources

A plan may include a `sources` mapping from codec identifiers to URIs —
suggested locations from which codec modules can be fetched. These are
advisory. The orchestrator is solely responsible for module resolution: it may
use the plan's sources, ignore them in favor of locally cached or
user-configured modules, or reject them based on policy.

```
Sources {
    [codec_id: string]: string,     // maps codec_id to fetch URI
}
```

The `sources` mapping serves the same role as the `src` field on pipeline steps
(consolidated to the top level), providing a single location for all module
fetch URIs. Content hash verification, if desired, is handled at the module
resolution layer, not in the plan format.

### Pipelines

A plan may define codec pipelines inline. Each pipeline follows the codec
pipeline schema (defined separately) and is keyed by a `codec_id` that is local
to the plan. Codec nodes reference these pipelines by `codec_id`, just as they
would reference an externally resolved codec.

```
Pipelines {
    [codec_id: string]: PipelineDefinition,
}
```

Inline pipeline definitions enable format drivers to construct multi-step codec
pipelines tailored to the data being processed, without requiring those
pipelines to be pre-registered in an external registry. When many nodes share
the same pipeline (e.g., all chunks in an array use the same encoding), the
pipeline is defined once and referenced from each node.

The `sources` mapping applies to codecs referenced within pipeline steps — the
pipeline's step `codec_id` values are resolved through the same module
resolution mechanism as top-level codec nodes.

### Error Policy

The plan declares an error policy that governs how the orchestrator handles
node failures during execution.

```
ErrorPolicy {
    mode: enum { Strict, PartialFailure },
    max_retries_storage: uint,     // retry count for transient storage errors
    retry_backoff_ms: uint,        // base backoff duration for retries
}
```

In `Strict` mode, any node failure causes the entire plan execution to abort.
In `PartialFailure` mode, failed nodes propagate failure to their dependents,
but independent subgraphs continue executing. The caller receives partial
results alongside error reports.

The error policy may be overridden by the caller at plan submission time. If no
policy is specified in the plan or by the caller, the default is `Strict`.

## Execution Model

### Plan Lifecycle

A plan progresses through four phases:

1. **Submission.** The caller provides a plan (serialized or constructed via
   API) and any external inputs. The orchestrator accepts the plan for
   processing.
2. **Validation.** The orchestrator validates the plan's structural integrity
   and resolves all module references. Validation includes: verifying the DAG
   is acyclic; verifying all edges connect compatible port types; resolving
   each codec URI to an available module and verifying that the module's
   signature matches the plan's port and parameter declarations; resolving
   driver callback references to exports on the format driver module; verifying
   all required parameters are present and correctly typed; and applying
   resource limit checks (maximum node count, maximum total read bytes, etc.).
   Validation may require fetching and compiling modules. If validation fails,
   the plan is rejected with a diagnostic error before any execution occurs.
3. **Execution.** The orchestrator executes the plan's nodes according to the
   schedule derived from the DAG structure. Execution is described in detail
   below.
4. **Completion.** The orchestrator reports the execution result to the caller:
   status (complete, partial failure, total failure), output data, and any
   error reports.

### Scheduling

The orchestrator derives an execution schedule from the plan's DAG. The
fundamental rule is: a node may execute once all of its input-providing nodes
have completed. Within this constraint, the orchestrator maximizes concurrency
— independent nodes (those with no data dependency between them) may execute in
parallel.

The orchestrator classifies nodes by scheduling category:

- **I/O nodes** (`read`, `write`) are dispatched to an asynchronous I/O
  subsystem. The orchestrator issues multiple concurrent reads to maximize
  storage throughput. Write nodes respect ordering constraints declared in
  their parameters.
- **Compute nodes** (`codec`, `driver_callback`, `scan`) are dispatched to a
  thread pool. The orchestrator maintains a pool of worker threads, each
  capable of invoking module functions.
- **Structural nodes** (`concat`, `slice`, `validate`, `choice`) are
  lightweight operations that the orchestrator may execute inline on the
  scheduling thread or on a worker, depending on cost.

The orchestrator is free to employ any scheduling strategy that respects the
DAG's dependency ordering. Possible strategies include depth-first (minimizes
peak memory by completing one chunk's pipeline before starting the next),
breadth-first (maximizes concurrency), or memory-aware scheduling (tracks live
buffer sizes and prioritizes nodes whose completion frees the most memory). The
choice of strategy is an implementation concern and is not prescribed by this
specification.

### Memory Management

The orchestrator manages all buffer allocation. Modules do not allocate their
own storage for inputs or outputs; they operate on buffers provided by the
orchestrator.

#### Buffer Space

The orchestrator maintains a buffer space for data flowing between plan nodes.
For each edge in the plan, the orchestrator allocates a buffer to hold the data
produced by the source node and consumed by the destination node. Buffers are
freed when all consumers have completed.

The orchestrator may pre-plan buffer allocation for nodes with predictable
output sizes (e.g., shuffle, where output size equals input size). For nodes
with unpredictable output sizes (e.g., entropy decoders), the orchestrator
provides an initial buffer and supports dynamic growth through the module
interface.

#### Data Movement Between Tiers

The orchestrator manages data movement between module memory spaces according
to the tiers of adjacent modules in the pipeline:

**Native → Native.** The orchestrator passes pointers into its buffer space
directly. Both modules operate on the same memory. Zero copies.

**Native → Core ABI.** The orchestrator writes data directly into the core ABI
module's linear memory (which the orchestrator created and controls). The
native module may write its output directly into the target module's linear
memory. Zero copies in the best case.

**Core ABI → Native.** The native module reads directly from the core ABI
module's linear memory. Zero copies.

**Core ABI → Core ABI.** The orchestrator copies data from the source module's
linear memory into the destination module's linear memory. At most one copy.

**Any → Component Model.** Data is passed through the canonical ABI, which
lifts from the source representation and lowers into the component's internal
memory. The orchestrator does not control the component's internal allocation.
Two copies (one lift, one lower).

**Component Model → Any.** Data is lifted from the component's internal
representation and lowered into the orchestrator's buffer space or the next
module's memory. Two copies.

The orchestrator optimizes data movement where possible. For example, when a
native module's output feeds a core ABI module's input, the orchestrator may
direct the native module to write directly into the core ABI module's linear
memory, avoiding any intermediate buffer.

The tier of each module is resolved during plan validation. The plan itself is
tier-agnostic — it references modules by URI, and the orchestrator resolves
each to the best available implementation. The same plan executes correctly
regardless of which tiers are used.

### Concurrent Chunk Processing

Many data processing tasks involve applying the same pipeline to many
independent chunks. The plan expresses this as independent subgraphs — each
chunk has its own read, decode, and output nodes, with no edges between chunks.

The orchestrator exploits this independence for concurrency. Independent
subgraphs execute in parallel, limited by the size of the thread pool and
available memory. The orchestrator issues storage reads for multiple chunks
concurrently, and begins decoding a chunk as soon as its read completes, while
other chunks are still being fetched.

For each concurrent chunk in flight, the orchestrator pre-allocates
non-overlapping buffer regions (for tiers where it manages memory). The peak
memory usage is bounded by the number of concurrent chunks multiplied by the
per-chunk buffer requirement.

### Streaming

The orchestrator supports two data transfer modes between nodes:

**Batch mode.** The source node produces its complete output before the
destination node begins. This is the default and is required for nodes that
need random access to their full input (e.g., shuffle, bitshuffle, transpose).

**Streaming mode.** The source node produces output incrementally, and the
destination node consumes it incrementally. This allows pipelining within a
single chunk's processing, reducing peak memory and latency. Streaming is only
possible when both the source node's output and the destination node's input
support incremental processing.

Each codec module advertises whether it supports streaming input, streaming
output, both, or neither. The orchestrator uses this information to determine
which edges in the plan can operate in streaming mode. If either the producer
or consumer requires batch semantics, the edge operates in batch mode.

The streaming protocol uses a channel-based interface inspired by Go's channel
model. Each streaming edge becomes a channel — an opaque pipe provided by the
orchestrator as imported functions. The codec reads from input channels and
writes to output channels using a simple blocking interface. The codec does not
know or care how the channel is implemented; the orchestrator provides the
channel semantics. The channel interface is defined in the Streaming Codec
Interface section under Module Interfaces.

Streaming mode is an optimization. The orchestrator may choose to use batch
mode for all edges without affecting correctness.

#### Streaming: Applicability and Tradeoffs

At realistic chunk sizes (sub-megabyte to low single-digit megabytes),
streaming within a single chunk's pipeline is unlikely to improve throughput.
The coordination overhead of suspending and resuming codec execution, managing
channel buffers, and scheduling producer-consumer handoffs may negate any
benefit from reduced peak memory. The primary throughput lever at typical chunk
sizes is chunk-level parallelism — processing many independent chunks
concurrently — which the batch execution model already supports.

Streaming becomes advantageous when individual chunks are large (roughly 20MB
or above), where the peak memory reduction from not materializing full
intermediate buffers is meaningful, and where the coordination overhead is
amortized over enough data to be negligible.

However, this RFC takes the position that data producers should be encouraged
toward smaller chunk sizes, where batch execution is both simpler and
sufficient. The streaming interface is specified for completeness and to
support edge cases (legacy data with large chunks, formats where chunk sizes
are not under the producer's control), but implementations should default to
batch execution and treat streaming as an opt-in optimization for large-chunk
workloads. It is entirely acceptable for an initial implementation to not
support streaming at all.

### Module Lifecycle

**Compiled modules** are cached globally by the orchestrator, keyed by content
hash. Compilation (parsing and compiling Wasm bytecode to native code) is
expensive and is performed at most once per module binary. The compiled module
cache persists for the lifetime of the orchestrator.

**Module instances** are pooled per worker thread. Each worker thread maintains
a pool of module instances for each module it has used. Instances are reused
across invocations to amortize instantiation cost and preserve internal
allocations (e.g., decompression contexts). The module contract requires that
instances behave identically across invocations — no observable state carries
over.

**Format driver instances** are tied to a specific file operation. A format
driver instance is created when a file is opened, persists through metadata
parsing, plan construction, and plan execution (for driver callbacks), and is
destroyed when the operation completes.

**Storage driver instances** are tied to a specific storage location. Their
lifecycle mirrors format driver instances.

### Module Resolution

The orchestrator resolves codec URIs to module implementations. Resolution
follows a precedence order:

1. User-configured module overrides (explicit mappings from URI to module binary).
2. Locally cached modules (previously fetched and verified).
3. Module hints from the plan (advisory URIs and content hashes).
4. Registry lookup (querying a codec registry by URI).

The orchestrator verifies module integrity (content hash) and compatibility
(signature matches plan requirements) before using a resolved module.

Format drivers and storage drivers are user-managed — they are explicitly
installed and configured, not fetched on demand. This reflects their higher
trust requirements: format drivers direct the orchestrator's execution, and
storage drivers access external resources. User curation provides a degree of
safety through friction and review.

Codecs may be fetched on demand by URI. Their blast radius is tightly
constrained by the sandbox and resource limits, making dynamic fetching
acceptable.

### Resource Limits

The orchestrator enforces resource limits to protect against malicious or buggy
modules and pathological plans.

**Memory limits.** A maximum total memory budget for plan execution. The
orchestrator rejects buffer allocation requests that would exceed this budget.

**I/O limits.** Maximum number of storage read operations and maximum total
bytes read. Prevents a malicious plan from issuing excessive I/O.

**Computation limits.** For WebAssembly modules, fuel-based execution metering
limits the number of instructions a module may execute per invocation. The fuel
budget is proportional to the input data size, with a configurable multiplier.
Execution traps if fuel is exhausted.

**Plan complexity limits.** Maximum DAG depth, maximum node count, maximum
fan-out per node. Prevents pathological plans that would overwhelm the
scheduler.

**Output size limits.** The orchestrator may reject codec outputs that are
disproportionately large relative to their inputs. The acceptable ratio is
configurable and may differ by codec type (e.g., decompressors may have a
higher allowance than encoders).

Resource limits are configurable by the caller and may have system-wide
defaults. The orchestrator reports resource exhaustion as a specific error
category.

## Module Interfaces

All modules — codecs, format drivers, and storage drivers — implement a uniform
calling convention regardless of tier. The tier determines how data crosses the
module boundary, not how the module's functionality is invoked.

### Calling Convention

Modules are invoked through a descriptor-based convention. The orchestrator
prepares a call descriptor containing references to all input data, output
regions, and parameters, then invokes the module's entry point with a reference
to this descriptor. For native modules, the descriptor contains pointers. For
core ABI modules, it contains offsets within the module's linear memory. For
Component Model modules, the descriptor is translated to canonical ABI types.

#### I/O Table

The I/O table is an array of port descriptors, ordered by the module's
signature: inputs first (in signature order), then outputs (in signature
order).

```
IOTable {
    entry_count: u32,
    entries: [IOEntry; entry_count],
}

IOEntry {
    data_ptr: u32,           // pointer/offset to data region
    data_len: u32,           // for inputs: byte count; for outputs: capacity
    actual_len_ptr: u32,     // for outputs: location where module writes actual bytes produced
}
```

#### Parameter Block

The parameter block contains scalar parameters and short strings, ordered by
the module's signature.

```
ParamBlock {
    count: u32,
    entries: [ParamEntry; count],
}

ParamEntry {
    type_tag: u8,            // uint=0, int=1, float=2, bool=3, string=4
    value_len: u32,
    value: [u8; value_len],  // little-endian encoded scalar, or UTF-8 string bytes
}
```

### Codec Module Interface

A codec module implements the following operations:

#### Discovery

**`signature`** — Returns the codec's signature descriptor: codec URI, input
port definitions, output port definitions, parameter definitions, and
capability flags (streaming support, output size estimation). The descriptor is
a serialized structure whose format is defined by the codec manifest
specification (out of scope for this RFC). The orchestrator reads the signature
once at module load time for plan validation.

#### Execution

**`decode(io_table, params) → status`** — Decodes data according to the codec's
algorithm. Reads from input ports and writes to output ports as defined in the
I/O table. Returns a status code: zero or positive for success (indicating
bytes written for single-output codecs), negative for error.

**`encode(io_table, params) → status`** — Encodes data according to the codec's
algorithm. Same convention as `decode`.

#### Optional

**`estimate_output_size(input_size) → estimated_size`** — Returns an estimate
of the output size for a given input size, or zero if estimation is not
possible. The orchestrator uses this for buffer pre-allocation.

#### Imports (provided by orchestrator)

**`request_output_buffer(port_index, min_additional) → (ptr, capacity)`** —
Called by the codec when it needs more output space than initially provided.
The orchestrator allocates or grows the output buffer and returns the new base
pointer and total capacity. Existing data is preserved at the new location. The
codec must use the returned pointer as the new output base; previous pointers
may be invalidated.

**`report_error(code, message_ptr, message_len)`** — Called by the codec to
report an error with a machine-readable code and human-readable message. After
calling this, the codec should return a negative status from the current
operation.

#### Streaming Codec Interface

The streaming interface is optional. A codec that supports streaming exports an
additional entry point and imports channel operations. A codec that does not
support streaming simply omits these exports; the orchestrator falls back to
batch invocation. Any codec that supports streaming must also support the batch
interface.

##### Streaming Export

**`stream(params_ptr, params_len, in_channels_ptr, in_channels_len,
out_channels_ptr, out_channels_len) → status`** — Executes the codec as a
streaming processor. The codec receives channel IDs for its input and output
ports (in signature order), reads incrementally from input channels, and writes
incrementally to output channels. The function runs until all input channels
are closed (indicating end of data), then flushes any buffered output, closes
its output channels, and returns.

The codec's implementation is a natural loop:

```
fn stream(params, in_channels, out_channels) {
    let input = in_channels[0];
    let output = out_channels[0];
    let ctx = init_decompressor(params);
    let buf = [0u8; CHUNK_SIZE];

    loop {
        let n = channel_read(input, buf.ptr, buf.len);
        if n == 0 { break; }         // input closed, no more data
        if n < 0 { return n; }        // error

        let result = process(ctx, buf[..n]);
        let status = channel_write(output, result.ptr, result.len);
        if status < 0 { return status; }  // downstream error
    }

    let remaining = flush(ctx);
    if remaining.len > 0 {
        channel_write(output, remaining.ptr, remaining.len);
    }
    channel_close(output);
    return SUCCESS;
}
```

All state (decompression contexts, internal buffers) is local to the function.
Cleanup happens naturally when the function returns. No separate init/free
lifecycle management is needed.

##### Streaming Imports (provided by orchestrator)

**`channel_read(channel_id, dest_ptr, min_bytes, max_bytes) → bytes_read`** —
Reads data from a channel into the specified buffer. The channel guarantees
that if it returns a positive value, at least `min_bytes` bytes have been read
(unless the stream ended with fewer bytes remaining, in which case the
remaining bytes are returned as a short read). The channel reads up to
`max_bytes` if that much data is available. Returns 0 if the channel has been
closed by the producer (end of stream), or a negative value on error. This call
may suspend the codec's execution until sufficient data is available.

The `min_bytes` parameter lets each codec express its framing needs on every
call. A shuffle codec reads in multiples of its element size: `channel_read(id,
buf, element_size * N, buf_cap)`. An RLE decoder reads a header byte:
`channel_read(id, buf, 1, 1)`, parses the run length, then reads the run body:
`channel_read(id, buf, run_length, run_length)`. A zstd decoder accepts any
amount of input: `channel_read(id, buf, 1, buf_cap)`.

**`channel_write(channel_id, src_ptr, src_len) → status`** — Writes data to a
channel from the specified buffer. Returns the number of bytes written
(positive) or a negative value on error. This call may suspend the codec's
execution if the downstream consumer is not ready to accept data
(backpressure). The orchestrator resumes the codec when buffer space becomes
available.

**`channel_close(channel_id)`** — Signals that the codec will not write any
more data to this channel. Downstream consumers will receive 0 from subsequent
reads after consuming any buffered data.

Channel IDs are assigned by the orchestrator and are opaque to the codec. They
are passed in port-signature order, matching the same port ordering as the
batch io_table.

##### Error Propagation Through Channels

If a producer codec encounters an error mid-stream, it calls `report_error` and
returns a negative status. The orchestrator catches this, forcibly closes all
downstream channels with an error state, and propagates the failure through the
plan DAG using the standard error propagation model. The consumer's next
`channel_read` returns a negative value, signaling that the upstream has
failed. The consumer should clean up and return an error status.

All output produced by a streaming session that ends in error is discarded by
the orchestrator. Partial decompressed data from a corrupt stream is not
delivered to downstream nodes.

##### Orchestrator Implementation Notes

The channel interface is deliberately abstract — the orchestrator decides how
to implement channel semantics. Several strategies are viable:

**Coroutine-based (single-threaded).** The orchestrator runs the producer and
consumer as coroutines on the same worker thread. When the producer calls
`channel_write`, the orchestrator suspends the producer and resumes the
consumer. When the consumer's `channel_read` finds the buffer empty, the
orchestrator suspends the consumer and resumes the producer. This requires only
a small intermediate buffer (one chunk's worth of data) and no thread
synchronization. Wasmtime's async execution mode supports this pattern — host
import functions can return futures that suspend the Wasm call stack.

**Thread-per-codec.** Each streaming codec runs on its own thread. Channels are
bounded queues with blocking read/write operations. This is the simplest to
implement and provides natural parallelism, but uses more threads and requires
thread-safe queue implementations.

**Hybrid.** Coroutines within a single chunk's pipeline (one worker thread
alternates between producer and consumer), with thread-level parallelism across
chunks (different chunks on different worker threads). This matches the
chunk-level parallelism model already described in the execution model.

The coroutine-based approach is recommended for initial implementations, as it
avoids thread synchronization overhead. Wasm async execution support is
required in the runtime for core ABI modules — Wasmtime provides this today.
Native modules can use platform coroutine or fiber mechanisms. Component Model
modules may use whatever async primitives the Component Model supports; precise
support for this tier is not yet specified and should be treated as a runtime
capability that may or may not be available.

A format driver module implements operations for both the planning phase and
the execution phase.

#### Planning Phase

**`parse_metadata() → status`** — Parses format metadata from the associated
storage source. The format driver calls storage read imports to fetch metadata
byte ranges, interprets them according to the format specification, and retains
the parsed metadata internally for subsequent plan construction. Returns a
status code.

**`build_read_plan(request, plan_buffer) → plan_length`** — Constructs a plan
for reading specified data from the file. The request describes which data to
    read (column indices, row ranges, or equivalent format-specific selectors).
    The format driver writes a serialized plan into the provided buffer and
    returns its length. If the buffer is too small, returns the required size
    as a negative value.

**`build_write_plan(schema, options, plan_buffer) → plan_length`** — Constructs
a plan for writing data into the format. The schema describes the data
structure. The options specify codec selection, chunk size hints, and other
format-specific encoding parameters. Same return convention as
`build_read_plan`.

#### Execution Phase (Callbacks)

Any function exported by the format driver module may be referenced as a
`driver_callback` node in a plan. These functions follow the same calling
convention as codec operations: they receive an I/O table and parameter block,
and return a status code. Common callback functions include:

- Dictionary or symbol table construction from source data
- Structural assembly (e.g., combining decoded columns into a record batch,
  Dremel reassembly)
- Statistics collection (min, max, null count, distinct count)
- Metadata finalization (aggregating per-chunk metadata into file-level
  metadata)

**`finalize_metadata(chunk_metadata, output_buffer) → length`** — After all
chunks have been encoded and written, aggregates per-chunk metadata (offsets,
sizes, statistics) into the file's metadata structure (e.g., Parquet footer,
Zarr metadata document). The orchestrator calls this after all write nodes have
completed.

#### Imports (provided by orchestrator)

**`storage_read(offset, dest_ptr, length) → bytes_read`** — Reads bytes from
the file's storage source into the specified memory location. Used during
metadata parsing.

**`storage_size() → size`** — Returns the total size of the storage source in
bytes.

**`report_error(code, message_ptr, message_len)`** — Same as codec error
reporting.

### Storage Driver Module Interface

A storage driver module abstracts over a specific storage backend.

#### Operations

**`open(uri) → handle`** — Opens a storage location identified by a URI.
Returns a handle for subsequent operations. The URI scheme determines the
storage backend (e.g., `file://`, `s3://`, `gs://`, `http://`).

**`read(handle, offset, dest_ptr, length) → bytes_read`** — Reads bytes from
the storage location at the given offset into the specified memory location.
Returns the number of bytes actually read (which may be less than requested at
end-of-source).

**`size(handle) → size`** — Returns the total size of the storage object in
bytes.

**`write(handle, offset, src_ptr, length) → bytes_written`** — Writes bytes to
the storage location at the given offset from the specified memory location.

**`close(handle)`** — Releases resources associated with the handle.

#### Imports (provided by orchestrator)

Storage drivers import host I/O primitives appropriate to their backend:

**`host_file_open(path) → fd`**, **`host_file_read(fd, offset, dest_ptr,
length) → bytes_read`**, **`host_file_write(fd, offset, src_ptr, length) →
bytes_written`**, **`host_file_size(fd) → size`**, **`host_file_close(fd)`** —
Host filesystem operations.

**`host_http_request(method, url, headers, body, dest_ptr, dest_cap) →
response_length`** — Host HTTP operations for cloud storage backends.

**`report_error(code, message_ptr, message_len)`** — Same as codec error
reporting.

The set of host I/O primitives is extensible. A storage driver declares which
host capabilities it requires, and the orchestrator verifies that it can
provide them before instantiation.

### Interface Uniformity Across Tiers

The logical interface is identical across all three tiers. The differences are
purely mechanical:

**Native modules** receive the I/O table and parameter block as pointers in the
host's address space. Data pointers in the I/O table point directly into the
orchestrator's buffer space (for inputs) or into pre-allocated output regions
(for outputs). Function calls are direct native function invocations (e.g.,
through a C ABI or Rust trait object).

**Core ABI modules** receive the I/O table and parameter block as offsets
within their linear memory. The orchestrator writes the I/O table, parameter
block, and input data into the module's linear memory before invocation, and
reads output data from the module's linear memory after invocation. Function
calls are Wasm function invocations through the runtime.

**Component Model modules** receive inputs and return outputs through the
canonical ABI. The orchestrator translates the I/O table concept into Component
Model function parameters (typed lists of byte arrays). The component's
internal memory management is opaque to the orchestrator.

The orchestrator encapsulates this translation. From the perspective of plan
execution, a node is simply "invoke module X with these inputs and parameters"
— the tier-specific mechanics are handled internally.

## Error Model

### Error Categories

Errors are classified into categories that determine handling behavior:

**Validation errors** are detected during plan validation, before execution
begins. These include invalid DAG structure (cycles, disconnected nodes),
incompatible port types on edges, unresolvable codec URIs, missing required
parameters, and resource limit violations. Validation errors reject the entire
plan.

**Storage errors** occur during I/O operations: network timeouts,
file-not-found, permission denied, read errors. These are potentially
transient. The orchestrator retries storage errors according to the plan's
error policy (`max_retries_storage` with exponential backoff). After retries
are exhausted, the node is marked failed.

**Codec errors** occur during data transformation: invalid compressed data,
unexpected end of stream, unsupported codec configuration. These indicate data
corruption or incorrect codec application. Retrying will not help. The node is
marked failed immediately.

**Integrity errors** are a specific subtype of codec errors: checksum
mismatches detected by `validate` nodes. The node is marked failed immediately.

**Resource exhaustion errors** occur when a module exceeds its resource budget:
fuel exhausted, output size limit exceeded, memory budget exceeded. The node is
marked failed immediately.

**Driver errors** occur in format driver or storage driver operations:
unparseable metadata, unsupported format version, internal driver failures.
During the planning phase, these reject the plan. During execution (in
callbacks), they fail the node.

### Failure Propagation

When a node fails, failure propagates forward through the DAG. All nodes that
transitively depend on the failed node (through edges) are marked failed
without being executed. Independent nodes — those with no dependency path to
the failed node — are unaffected and continue executing.

For fan-in nodes (nodes with multiple input ports fed by different upstream
subgraphs), the behavior depends on the error policy:

- In **Strict** mode, the fan-in node fails if any of its inputs failed. This
  cascades to all nodes downstream of the fan-in, effectively aborting the
  entire plan.
- In **PartialFailure** mode, the fan-in node receives explicit information
  about which inputs are available and which failed. It may proceed with
  partial inputs if it is capable of producing a meaningful partial result. If
  it cannot (e.g., a `concat` missing a middle chunk), it fails.

The `driver_callback` nodes used for assembly operations must handle partial
inputs in PartialFailure mode: they receive a bitmask indicating which input
ports carry valid data. The format driver decides how to handle gaps (e.g.,
filling with nulls, excluding missing chunks, or failing).

### Error Reporting

Each failed node produces a structured error report:

```
NodeError {
    node_id: u32,
    category: enum { Storage, Codec, Integrity, ResourceExhaustion, Driver },
    code: i32,                  // machine-readable error code
    message: string,            // human-readable description
    retryable: bool,            // whether the error is transient
    retries_attempted: uint,    // number of retries before failure (storage errors)
}
```

The plan execution result aggregates all errors:

```
ExecutionResult {
    status: enum { Complete, PartialFailure, TotalFailure },
    outputs: Map<string, BufferRef>,     // named plan outputs with valid data
    failed_outputs: [string],            // named plan outputs that are missing due to failures
    errors: [NodeError],                 // all errors encountered during execution
}
```

- **Complete**: all plan outputs are available, no errors.
- **PartialFailure**: some plan outputs are available, some failed. Only
  possible in PartialFailure error policy mode.
- **TotalFailure**: no plan outputs are available. Either Strict mode with any
  failure, or PartialFailure mode where failures cascaded to all outputs.

### Error Policy Configuration

The error policy is resolved from, in order of precedence:

1. Per-plan-submission override provided by the caller.
2. Error policy declared in the plan.
3. System-wide default (Strict).

## Observability

### Plan Execution Tracing

The orchestrator emits structured trace events during plan execution. Tracing
is opt-in; when enabled, events are written to a pre-allocated trace buffer to
avoid allocation overhead during execution.

#### Trace Events

```
TraceEvent {
    timestamp_ns: u64,
    node_id: u32,
    event_type: enum {
        NodeScheduled,
        NodeStarted,
        NodeCompleted,
        NodeFailed,
        BufferAllocated,
        BufferGrown,
        BufferFreed,
        IODispatched,
        IOCompleted,
        ModuleLoaded,
        ModuleCacheHit,
    },
    duration_ns: u64,           // for Completed/Failed events
    bytes_in: u64,              // total input bytes for the node
    bytes_out: u64,             // total output bytes for the node
    memory_high_water: u64,     // peak memory usage at this point in execution
    detail: string,             // optional: error message, module URI, etc.
}
```

Trace events enable:

- **Performance analysis.** Identifying bottlenecks: which nodes are slowest,
  whether the pipeline is I/O-bound or compute-bound, and where memory pressure
  is highest.
- **Debugging.** Reconstructing the execution timeline to diagnose failures:
  what ran before the failing node, what its inputs looked like, and how the
  failure propagated.
- **Capacity planning.** Understanding memory usage patterns to tune buffer
  sizes, concurrency limits, and resource budgets.

#### Trace Output

The trace may be consumed in multiple ways:

- **Post-execution dump.** The caller reads the trace buffer after plan
  completion. Suitable for batch analysis and test diagnostics.
- **Live streaming.** The orchestrator streams trace events to a subscriber
  during execution. Suitable for monitoring dashboards and live debugging.
- **Structured export.** The trace can be serialized to standard formats (e.g.,
  Chrome Trace Event Format, OpenTelemetry spans) for analysis in external
  tools.

### System Metrics

The orchestrator tracks aggregate metrics across plan executions:

- **Module cache**: compiled module count, cache hit/miss rates, total
  compilation time.
- **Instance pools**: instances created, reused, peak pool sizes per module.
- **Memory**: current usage, high-water mark, allocation/free counts, growth
  events.
- **I/O**: total bytes read/written, request counts, latency distributions per
  storage driver.
- **Execution**: plans executed, nodes executed, error counts by category, plan
  durations.

System metrics are a host-side concern. The orchestrator exposes them through a
metrics interface suitable for integration with standard observability systems
(Prometheus, OpenTelemetry, StatsD, etc.). The specific integration mechanism
is an implementation concern.

### Codec Transparency

Codecs do not emit trace events directly. The orchestrator instruments all
codec invocations externally — it measures timing, input/output sizes, and
errors around each call without the codec's cooperation. This keeps the codec
interface minimal and avoids requiring trace infrastructure in every codec
implementation.

If finer-grained visibility into a specific codec's behavior is needed (e.g.,
profiling block-level decompression within zstd), an instrumented wrapper
module can be used. The wrapper delegates to the actual codec and emits
additional trace events through a custom import. This approach avoids burdening
the standard codec interface with observability concerns.

### Plan Visualization

Plans are DAGs and benefit from visual representation for debugging and
documentation. The serialized plan format contains sufficient information to
generate graph visualizations (e.g., Graphviz DOT format) without access to the
orchestrator or any module implementations. External tools can parse the plan
and render the DAG, annotated with node types, codec URIs, port names, and
parameter values.

When combined with a trace, a visualization tool can overlay execution data —
timing, data sizes, and error status — onto the plan graph, providing a
complete picture of what was planned and what happened during execution.

## Examples

The following examples illustrate plans in a JSON-like notation for
readability. The canonical serialization is FlatBuffers; this notation is for
human consumption and tooling interoperability only. Node IDs are assigned
sequentially for clarity.

### Example 1: Simple Chunk Decode

A single chunk stored with shuffle followed by zstd compression. The decode
pipeline is `zstd → shuffle → output`. The plan defines the pipeline once and
invokes it from a single codec node.

```json
{
  "version": "0.1.0",
  "error_policy": { "mode": "strict", "max_retries_storage": 3, "retry_backoff_ms": 100 },

  "sources": {
    "zstd": "https://registry.cylf.dev/codecs/zstd/1.0.0.wasm",
    "shuffle": "https://registry.cylf.dev/codecs/shuffle/1.0.0.wasm"
  },

  "pipelines": {
    "zarr-f32-decode": {
      "codec_id": "zarr-f32-decode",
      "direction": "decode",
      "inputs": {
        "bytes": { "type": "bytes" }
      },
      "constants": {
        "element_size": { "type": "uint", "value": 4 }
      },
      "outputs": {
        "bytes": "unshuffle.bytes"
      },
      "steps": {
        "decompress": {
          "codec_id": "zstd",
          "inputs": { "bytes": "input.bytes" }
        },
        "unshuffle": {
          "codec_id": "shuffle",
          "inputs": {
            "bytes": "decompress.bytes",
            "element_size": "constant.element_size"
          }
        }
      }
    }
  },

  "inputs": [],
  "outputs": [
    { "name": "decoded_chunk", "type": "bytes", "node_id": 1, "port_index": 0 }
  ],

  "nodes": [
    {
      "id": 0,
      "type": "read",
      "params": {
        "source": "file:///data/array.zarr/c.0.0",
        "offset": 0,
        "length": 16384
      }
    },
    {
      "id": 1,
      "type": "codec",
      "codec_id": "zarr-f32-decode",
      "direction": "decode",
      "input_bindings": {
        "bytes": { "from_node": 0, "from_port": 0 }
      }
    }
  ],

  "edges": [
    { "from_node": 0, "from_port": 0, "to_node": 1, "to_port": 0 }
  ]
}
```

The plan is compact: two nodes (read, decode) with one edge between them. The
codec pipeline's internal structure (zstd → shuffle) is defined in the
`pipelines` section and is opaque to the plan DAG. The `element_size` parameter
is baked into the pipeline as a constant because it is uniform across all
chunks of this array. The orchestrator resolves `zstd` and `shuffle` to
available modules at any tier, consulting the `sources` mapping if needed.

### Example 2: Parquet Column Chunk with Dictionary Encoding

Reading a single dictionary-encoded column chunk containing two data pages. The
dictionary page is read once and shared across both data page decode pipelines.
This example shows fan-out (dictionary shared across pages), parallel
independent subgraphs (the two data pages), driver callbacks for assembly, and
pipeline reuse (both pages use the same decode pipeline).

The Parquet format driver defines the per-page decode pipeline in the plan's
`pipelines` section. This pipeline references the well-known codec
`parquet-v2-dict-decode` from the codec inventory, which internally composes
`parquet-page-v2-split`, `rle-parquet`, and `plain-dictionary`. Because this
pipeline is defined in the codec inventory, the format driver references it by
`codec_id` rather than inlining its steps.

```json
{
  "version": "0.1.0",
  "error_policy": { "mode": "partial_failure", "max_retries_storage": 3, "retry_backoff_ms": 100 },

  "sources": {
    "zstd": "https://registry.cylf.dev/codecs/zstd/1.0.0.wasm",
    "parquet-page-v2-split": "https://registry.cylf.dev/codecs/parquet-page-v2-split/1.0.0.wasm",
    "rle-parquet": "https://registry.cylf.dev/codecs/rle-parquet/1.0.0.wasm",
    "plain-dictionary": "https://registry.cylf.dev/codecs/plain-dictionary/1.0.0.wasm"
  },

  "pipelines": {
    "pq-dict-page-decode": {
      "codec_id": "pq-dict-page-decode",
      "direction": "decode",
      "inputs": {
        "data_page": { "type": "bytes" },
        "dict_page": { "type": "bytes" },
        "rep_length": { "type": "uint" },
        "def_length": { "type": "uint" },
        "index_bit_width": { "type": "uint" },
        "physical_type": { "type": "string" }
      },
      "constants": {
        "def_bit_width": { "type": "uint", "value": 1 },
        "rep_bit_width": { "type": "uint", "value": 1 }
      },
      "outputs": {
        "values": "dict_lookup.bytes",
        "def_levels": "decode_def.bytes",
        "rep_levels": "decode_rep.bytes"
      },
      "steps": {
        "decompress": {
          "codec_id": "zstd",
          "inputs": { "bytes": "input.data_page" }
        },
        "split": {
          "codec_id": "parquet-page-v2-split",
          "inputs": {
            "bytes": "decompress.bytes",
            "rep_length": "input.rep_length",
            "def_length": "input.def_length"
          }
        },
        "decode_indices": {
          "codec_id": "rle-parquet",
          "inputs": {
            "bytes": "split.value_bytes",
            "bit_width": "input.index_bit_width"
          }
        },
        "dict_lookup": {
          "codec_id": "plain-dictionary",
          "inputs": {
            "indices": "decode_indices.bytes",
            "dictionary": "input.dict_page",
            "physical_type": "input.physical_type"
          }
        },
        "decode_def": {
          "codec_id": "rle-parquet",
          "inputs": {
            "bytes": "split.def_bytes",
            "bit_width": "constant.def_bit_width"
          }
        },
        "decode_rep": {
          "codec_id": "rle-parquet",
          "inputs": {
            "bytes": "split.rep_bytes",
            "bit_width": "constant.rep_bit_width"
          }
        }
      }
    }
  },

  "inputs": [],
  "outputs": [
    { "name": "column_values", "type": "bytes", "node_id": 5, "port_index": 0 }
  ],

  "nodes": [
    {
      "id": 0,
      "type": "read",
      "comment": "Read dictionary page",
      "params": { "source": "s3://bucket/table.parquet", "offset": 1000, "length": 2048 }
    },
    {
      "id": 1,
      "type": "read",
      "comment": "Read data page 0",
      "params": { "source": "s3://bucket/table.parquet", "offset": 3048, "length": 8192 }
    },
    {
      "id": 2,
      "type": "codec",
      "comment": "Decode data page 0",
      "codec_id": "pq-dict-page-decode",
      "direction": "decode",
      "input_bindings": {
        "data_page": { "from_node": 1, "from_port": 0 },
        "dict_page": { "from_node": 0, "from_port": 0 },
        "rep_length": { "constant": 128 },
        "def_length": { "constant": 128 },
        "index_bit_width": { "constant": 8 },
        "physical_type": { "constant": "BYTE_ARRAY" }
      }
    },
    {
      "id": 3,
      "type": "read",
      "comment": "Read data page 1",
      "params": { "source": "s3://bucket/table.parquet", "offset": 11240, "length": 8192 }
    },
    {
      "id": 4,
      "type": "codec",
      "comment": "Decode data page 1",
      "codec_id": "pq-dict-page-decode",
      "direction": "decode",
      "input_bindings": {
        "data_page": { "from_node": 3, "from_port": 0 },
        "dict_page": { "from_node": 0, "from_port": 0 },
        "rep_length": { "constant": 128 },
        "def_length": { "constant": 128 },
        "index_bit_width": { "constant": 8 },
        "physical_type": { "constant": "BYTE_ARRAY" }
      }
    },
    {
      "id": 5,
      "type": "driver_callback",
      "comment": "Assemble decoded pages into column array with nullability",
      "function_name": "assemble_column",
      "ports_in": [
        { "name": "values_0", "type": "bytes" },
        { "name": "def_levels_0", "type": "bytes" },
        { "name": "values_1", "type": "bytes" },
        { "name": "def_levels_1", "type": "bytes" }
      ]
    }
  ],

  "edges": [
    { "from_node": 0, "from_port": 0, "to_node": 2, "to_port": 1 },
    { "from_node": 1, "from_port": 0, "to_node": 2, "to_port": 0 },

    { "from_node": 0, "from_port": 0, "to_node": 4, "to_port": 1 },
    { "from_node": 3, "from_port": 0, "to_node": 4, "to_port": 0 },

    { "from_node": 2, "from_port": 0, "to_node": 5, "to_port": 0 },
    { "from_node": 2, "from_port": 1, "to_node": 5, "to_port": 1 },
    { "from_node": 4, "from_port": 0, "to_node": 5, "to_port": 2 },
    { "from_node": 4, "from_port": 1, "to_node": 5, "to_port": 3 }
  ]
}
```

Key observations:

- **The plan DAG has 6 nodes.** The previous version had 15 nodes because it
  inlined the codec pipeline. Pipeline reuse collapses the complexity: the
  multi-step decode pipeline is defined once and invoked twice (nodes 2 and 4),
  with per-page parameters supplied via `input_bindings`.
- **Node 0** (dictionary read) fans out to both nodes 2 and 4. The dictionary
  bytes are passed as the `dict_page` input binding to each pipeline
  invocation.
- **Nodes 2 and 4** are independent — they can execute in parallel. Each
  invokes the same pipeline (`pq-dict-page-decode`) with different data page
  bytes and the shared dictionary.
- **Node 5** is a driver callback for column assembly. Its `ports_in` are
  declared explicitly because driver callbacks do not have pre-registered
  signatures — the format driver defines them at plan construction time. The
  output ports of codec nodes (2 and 4) are determined by the pipeline's
  `outputs` declaration: port 0 is `values`, port 1 is `def_levels`, port 2 is
  `rep_levels`.
- The repetition level outputs (port 2 of nodes 2 and 4) are unused in this
  non-nested column example. A nested column's plan would wire them to the
  assembly callback as additional inputs.
- Per-page parameters like `rep_length` and `def_length` are supplied as inline
  constants in the `input_bindings`. These values come from the Parquet page
  header, parsed by the format driver during metadata reading. If pages had
  different rep/def lengths, each node's `input_bindings` would carry different
  values while still sharing the same pipeline definition.

### Example 3: Encode with Dictionary Construction

Encoding a column of string data into Parquet format with dictionary encoding
and zstd compression. The plan has two phases: a scan phase that builds the
dictionary from the source data, and an encode phase that uses the dictionary
to encode and write data pages.

```json
{
  "version": "0.1.0",
  "error_policy": { "mode": "strict", "max_retries_storage": 3, "retry_backoff_ms": 100 },

  "sources": {
    "zstd": "https://registry.cylf.dev/codecs/zstd/1.0.0.wasm",
    "rle-parquet": "https://registry.cylf.dev/codecs/rle-parquet/1.0.0.wasm",
    "plain-parquet": "https://registry.cylf.dev/codecs/plain-parquet/1.0.0.wasm"
  },

  "pipelines": {
    "pq-dict-page-encode": {
      "codec_id": "pq-dict-page-encode",
      "direction": "encode",
      "inputs": {
        "indices": { "type": "bytes" },
        "index_bit_width": { "type": "uint" }
      },
      "constants": {
        "level": { "type": "int", "value": 3 }
      },
      "outputs": {
        "bytes": "compress.bytes"
      },
      "steps": {
        "rle_encode": {
          "codec_id": "rle-parquet",
          "inputs": {
            "bytes": "input.indices",
            "bit_width": "input.index_bit_width"
          }
        },
        "compress": {
          "codec_id": "zstd",
          "inputs": {
            "bytes": "rle_encode.bytes",
            "level": "constant.level"
          },
          "encode_only_inputs": ["level"]
        }
      }
    }
  },

  "inputs": [
    { "name": "source_data", "type": "bytes", "node_id": 0, "port_index": 0 }
  ],
  "outputs": [],

  "nodes": [
    {
      "id": 0,
      "type": "scan",
      "comment": "Build dictionary and produce indices from source data",
      "function_name": "build_dictionary",
      "ports_in": [ { "name": "bytes", "type": "bytes" } ],
      "ports_out": [
        { "name": "dictionary", "type": "bytes" },
        { "name": "indices", "type": "bytes" },
        { "name": "index_bit_width", "type": "uint" },
        { "name": "stats", "type": "bytes" }
      ]
    },
    {
      "id": 1,
      "type": "codec",
      "comment": "Encode dictionary page as PLAIN",
      "codec_id": "plain-parquet",
      "direction": "encode",
      "input_bindings": {
        "bytes": { "from_node": 0, "from_port": 0 },
        "physical_type": { "constant": "BYTE_ARRAY" }
      }
    },
    {
      "id": 2,
      "type": "codec",
      "comment": "RLE-encode and compress data page indices",
      "codec_id": "pq-dict-page-encode",
      "direction": "encode",
      "input_bindings": {
        "indices": { "from_node": 0, "from_port": 1 },
        "index_bit_width": { "from_node": 0, "from_port": 2 }
      }
    },
    {
      "id": 3,
      "type": "write",
      "comment": "Write dictionary page",
      "params": { "source": "file:///output/table.parquet", "offset": 1000, "ordering": 0 }
    },
    {
      "id": 4,
      "type": "write",
      "comment": "Write compressed data page",
      "params": { "source": "file:///output/table.parquet", "offset": 3048, "ordering": 1 }
    },
    {
      "id": 5,
      "type": "driver_callback",
      "comment": "Finalize column chunk and file metadata",
      "function_name": "finalize_metadata",
      "ports_in": [
        { "name": "stats", "type": "bytes" },
        { "name": "dict_page_bytes", "type": "bytes" },
        { "name": "data_page_bytes", "type": "bytes" }
      ],
      "ports_out": [ { "name": "footer", "type": "bytes" } ]
    },
    {
      "id": 6,
      "type": "write",
      "comment": "Write file footer",
      "params": { "source": "file:///output/table.parquet", "ordering": 2 }
    }
  ],

  "edges": [
    { "from_node": 0, "from_port": 0, "to_node": 1, "to_port": 0 },
    { "from_node": 0, "from_port": 1, "to_node": 2, "to_port": 0 },
    { "from_node": 0, "from_port": 2, "to_node": 2, "to_port": 1 },

    { "from_node": 1, "from_port": 0, "to_node": 3, "to_port": 0 },
    { "from_node": 2, "from_port": 0, "to_node": 4, "to_port": 0 },

    { "from_node": 0, "from_port": 3, "to_node": 5, "to_port": 0 },
    { "from_node": 1, "from_port": 0, "to_node": 5, "to_port": 1 },
    { "from_node": 2, "from_port": 0, "to_node": 5, "to_port": 2 },

    { "from_node": 5, "from_port": 0, "to_node": 6, "to_port": 0 }
  ]
}
```

Key observations:

- **Node 0** (scan) is a driver callback that reads the source data, builds a
  dictionary, produces indices, determines the optimal bit width for index
  encoding, and collects statistics. All outputs are separate ports. The DAG
  naturally enforces that encoding cannot begin until the scan completes.
- **Nodes 1 and 2** can execute in parallel — encoding the dictionary page
  (PLAIN) and encoding the data page indices (RLE + zstd) are independent once
  the scan outputs are available.
- **The `index_bit_width`** is a scan output wired to the encode pipeline's
  input, not a constant. The format driver computes it from the dictionary
  cardinality during the scan — this is an example of a value that varies per
  column chunk and must be a pipeline input rather than a constant.
- **Write nodes** (3, 4, 6) have explicit `ordering` constraints. The
  orchestrator writes the dictionary page before the data page, and the footer
  last.
- **Node 5** (metadata finalization) fans in from the scan's statistics, the
  encoded dictionary page, and the encoded data page. It needs the encoded
  bytes to determine their sizes for the file metadata. Its output (the Parquet
  footer) is written by node 6.

### Example 4: Choice Node for Conditional Codec Selection

A plan that reads a chunk whose compression codec is not known at planning time
— the format stores a codec identifier in the chunk header. The plan uses a
choice node to select between zstd and lz4 decoding at runtime.

```json
{
  "version": "0.1.0",
  "error_policy": { "mode": "strict", "max_retries_storage": 3, "retry_backoff_ms": 100 },

  "sources": {
    "zstd": "https://registry.cylf.dev/codecs/zstd/1.0.0.wasm",
    "lz4": "https://registry.cylf.dev/codecs/lz4/1.0.0.wasm",
    "shuffle": "https://registry.cylf.dev/codecs/shuffle/1.0.0.wasm"
  },

  "pipelines": {},

  "inputs": [],
  "outputs": [
    { "name": "decoded_chunk", "type": "bytes", "node_id": 3, "port_index": 0 }
  ],

  "nodes": [
    {
      "id": 0,
      "type": "read",
      "params": { "source": "file:///data/store/chunk.0", "offset": 0, "length": 32768 }
    },
    {
      "id": 1,
      "type": "driver_callback",
      "comment": "Parse chunk header, extract codec ID and payload",
      "function_name": "parse_chunk_header",
      "ports_in": [ { "name": "raw_chunk", "type": "bytes" } ],
      "ports_out": [
        { "name": "codec_id", "type": "string" },
        { "name": "payload", "type": "bytes" }
      ]
    },
    {
      "id": 2,
      "type": "choice",
      "comment": "Select decompressor based on codec ID from header",
      "discriminator": { "node_id": 1, "port_index": 0 },
      "input_bindings": {
        "bytes": { "from_node": 1, "from_port": 1 }
      },
      "branches": {
        "zstd": {
          "codec_id": "zstd",
          "direction": "decode"
        },
        "lz4": {
          "codec_id": "lz4",
          "direction": "decode"
        }
      }
    },
    {
      "id": 3,
      "type": "codec",
      "comment": "Unshuffle after decompression",
      "codec_id": "shuffle",
      "direction": "decode",
      "input_bindings": {
        "bytes": { "from_node": 2, "from_port": 0 },
        "element_size": { "constant": 8 }
      }
    }
  ],

  "edges": [
    { "from_node": 0, "from_port": 0, "to_node": 1, "to_port": 0 },
    { "from_node": 1, "from_port": 1, "to_node": 2, "to_port": 0 },
    { "from_node": 2, "from_port": 0, "to_node": 3, "to_port": 0 }
  ]
}
```

Key observations:

- **Node 1** parses the chunk header to extract both the codec identifier (a
  string) and the payload (the compressed bytes without the header). This is a
  format-specific operation implemented as a driver callback.
- **Node 2** is a choice node. The discriminator references node 1's `codec_id`
  output. At runtime, the orchestrator evaluates this value and selects the
  matching branch. Both branches have identical output signatures (`bytes →
  bytes`), satisfying the constraint that downstream wiring is
  branch-independent. The branches reference single codecs directly; for more
  complex conditional pipelines, branches could reference pipeline definitions
  from the `pipelines` section.
- **Node 3** (shuffle decode) runs after whichever branch was selected. It
  receives its `element_size` parameter as an inline constant in
  `input_bindings`, and its data from the choice node's output. The plan is
  branch-agnostic downstream of the choice.
- The orchestrator must resolve and validate modules for all branches during
  plan validation, even though only one branch will execute per invocation.
  This ensures that execution never fails due to an unresolvable codec in an
  unchosen branch.

### Example 5: Multi-Chunk Decode with Pipeline Reuse

Reading four chunks from a Zarr array where all chunks share the same encoding
(`shuffle → zstd`). This example demonstrates the primary motivation for
pipeline reuse: the pipeline is defined once, and each chunk is a compact node
with per-chunk input bindings.

```json
{
  "version": "0.1.0",
  "error_policy": { "mode": "partial_failure", "max_retries_storage": 3, "retry_backoff_ms": 100 },

  "sources": {
    "zstd": "https://registry.cylf.dev/codecs/zstd/1.0.0.wasm",
    "shuffle": "https://registry.cylf.dev/codecs/shuffle/1.0.0.wasm"
  },

  "pipelines": {
    "zarr-f32-decode": {
      "codec_id": "zarr-f32-decode",
      "direction": "decode",
      "inputs": {
        "bytes": { "type": "bytes" }
      },
      "constants": {
        "element_size": { "type": "uint", "value": 4 }
      },
      "outputs": {
        "bytes": "unshuffle.bytes"
      },
      "steps": {
        "decompress": {
          "codec_id": "zstd",
          "inputs": { "bytes": "input.bytes" }
        },
        "unshuffle": {
          "codec_id": "shuffle",
          "inputs": {
            "bytes": "decompress.bytes",
            "element_size": "constant.element_size"
          }
        }
      }
    }
  },

  "inputs": [],
  "outputs": [
    { "name": "chunk_0", "type": "bytes", "node_id": 1, "port_index": 0 },
    { "name": "chunk_1", "type": "bytes", "node_id": 3, "port_index": 0 },
    { "name": "chunk_2", "type": "bytes", "node_id": 5, "port_index": 0 },
    { "name": "chunk_3", "type": "bytes", "node_id": 7, "port_index": 0 }
  ],

  "nodes": [
    { "id": 0, "type": "read", "params": { "source": "file:///data/array.zarr/c.0", "offset": 0, "length": 16384 } },
    { "id": 1, "type": "codec", "codec_id": "zarr-f32-decode", "direction": "decode",
      "input_bindings": { "bytes": { "from_node": 0, "from_port": 0 } } },

    { "id": 2, "type": "read", "params": { "source": "file:///data/array.zarr/c.1", "offset": 0, "length": 16384 } },
    { "id": 3, "type": "codec", "codec_id": "zarr-f32-decode", "direction": "decode",
      "input_bindings": { "bytes": { "from_node": 2, "from_port": 0 } } },

    { "id": 4, "type": "read", "params": { "source": "file:///data/array.zarr/c.2", "offset": 0, "length": 16384 } },
    { "id": 5, "type": "codec", "codec_id": "zarr-f32-decode", "direction": "decode",
      "input_bindings": { "bytes": { "from_node": 4, "from_port": 0 } } },

    { "id": 6, "type": "read", "params": { "source": "file:///data/array.zarr/c.3", "offset": 0, "length": 16384 } },
    { "id": 7, "type": "codec", "codec_id": "zarr-f32-decode", "direction": "decode",
      "input_bindings": { "bytes": { "from_node": 6, "from_port": 0 } } }
  ],

  "edges": [
    { "from_node": 0, "from_port": 0, "to_node": 1, "to_port": 0 },
    { "from_node": 2, "from_port": 0, "to_node": 3, "to_port": 0 },
    { "from_node": 4, "from_port": 0, "to_node": 5, "to_port": 0 },
    { "from_node": 6, "from_port": 0, "to_node": 7, "to_port": 0 }
  ]
}
```

Key observations:

- **The pipeline is defined once.** All four chunk decode nodes reference
  `zarr-f32-decode`. For an array with 10,000 chunks, the plan would have
  10,000 read/decode node pairs but still only one pipeline definition.
- **All four read→decode pairs are independent.** The orchestrator can execute
  all eight nodes across four parallel tracks, limited only by thread pool size
  and memory budget. With `partial_failure` error policy, a corrupted chunk
  does not prevent the other three from decoding successfully.
- **Each codec node is minimal** — just a `codec_id`, `direction`, and
  `input_bindings`. The per-node overhead in the serialized plan is small,
  making this representation efficient even for large numbers of chunks.
- **The `element_size` constant is shared.** It is baked into the pipeline
  definition, not repeated in each node's `input_bindings`. If different chunks
  had different element sizes (e.g., a heterogeneous-type array),
  `element_size` would be a pipeline input instead of a constant, and each
  node's `input_bindings` would supply the appropriate value.

## Prior Art and References

- **Codec pipeline model.** The lower-level codec pipeline specification,
  defined separately, describes how codec operations compose into DAGs for
  transforming chunk data. Plans build on this concept.
- **cylf codec inventory.** The catalog of codec signatures, awareness
  categories, and pipeline decompositions that informs the codec interface
  design.
- **WebAssembly Core Specification.** Defines the execution model for core ABI
  modules: linear memory, function imports/exports, and instruction semantics.
- **WebAssembly Component Model.** Defines the canonical ABI, interface types,
  and component composition model used by Component Model tier modules.
- **FlatBuffers.** The serialization framework used for the plan format, chosen
  for zero-copy deserialization and cross-language support.
- **Apache Arrow.** Columnar memory format used as a reference for buffer
  layout, particularly for assembly operations that produce Arrow-compatible
  output.
- **TVM GraphPlanMemory.** Example of DAG-based memory planning in a dataflow
  system, where tensor lifetimes are analyzed to minimize peak memory through
  buffer reuse.

## Open Questions

1. **Plan format schema.** The exact FlatBuffers schema for the plan format
   needs to be defined. This RFC describes the logical structure; the binary
   schema is a follow-up deliverable.
2. **Codec manifest specification.** The format and content of codec signature
   descriptors (embedded as Wasm custom sections or served by registries) is
   referenced but not defined by this RFC.
3. **Streaming runtime requirements.** The streaming codec interface requires
   Wasm async execution support (the ability to suspend Wasm execution at
   import call boundaries). Wasmtime provides this today. Compatibility with
   other Wasm runtimes and with the Component Model tier needs verification.
4. **Encode-path multi-pass coordination.** The interaction between scan nodes,
   driver callbacks, and the main encode pipeline — particularly how the
   orchestrator coordinates multi-pass operations where a preparation pass must
5. **Plan composition.** Whether plans can reference other plans as sub-units
   (analogous to how codec pipelines can reference other pipelines) is
   deferred. The current design does not support plan nesting.
6. **Component Model canonical ABI mapping.** The precise mapping from the I/O
   table and parameter block convention to Component Model function signatures
   needs specification, possibly as a WIT definition.
7. **Versioning and compatibility.** Policies for plan format version
   evolution, codec signature compatibility across versions, and module binary
   compatibility need to be formalized.
