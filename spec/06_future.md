# Open Questions & Roadmap

This document consolidates unresolved design questions and planned work across
the Cylf ecosystem. Items are grouped by area. Items marked ✅ have been
resolved and are retained for context.

---

## Codec Signature Model

### Codec ID versioning

Codec identifiers are currently unversioned slugs (`zstd`, `shuffle`,
`tiff-predictor-2`). A versioned identifier scheme is needed to support
iterating on codec specifications without breaking existing pipelines. Open
questions include the format (e.g. `zstd@1.0.0`, a URI scheme, or something
else), what constitutes a breaking vs non-breaking change, and how version
resolution works when a pipeline references an unversioned identifier.

### JSON Schema adoption

The signature format could be formalized using JSON Schema rather than a
bespoke schema description. This would enable standard validation tooling,
editor support (autocomplete, inline errors), and cross-language code
generation. Worth evaluating whether JSON Schema is expressive enough for the
signature model's needs.

### `parquet-page-v2-split` direction semantics

It is unclear whether `rep_length` and `def_length` on `parquet-page-v2-split`
are truly external inputs supplied by the format driver, or whether they are
parsed from the page header bytes that are part of the codec's input data
(making the codec self-describing for those values). This needs verification
against the Parquet v2 spec before the codec's signature can be finalized.

### Self-describing codec boundary

Blosc is classified as atomic (self-describing) because its header is embedded
in the chunk. The criterion for when a codec is self-describing vs
externally-parameterized should be formalized — currently it is described by
example rather than by rule.

---

## Pipeline Model

### Conditional pipeline steps

Some codecs require conditional logic within a pipeline. ORC's varint encoding
applies a zigzag step only for signed types. Parquet pages may use different
compression codecs determined at runtime by a value in the page header (e.g.
the encoding type is part of the chunk data, not known at pipeline construction
time). The current pipeline model has no mechanism for conditional step
execution or branching.

The [RFC on Format Drivers and Data Orchestration](07_proposals/01_orchestration.md)
proposes a `choice` node at the plan level. Whether conditional branching
belongs in the codec pipeline model itself or only in the higher-level plan
format is not yet decided. This is critical for allowing formats like Parquet,
where encoding metadata is embedded in chunk data, to reuse generic codecs
rather than requiring format-specific monolithic decoders.

### Pipeline nesting

A pipeline is a codec, which implies it can appear as a step inside another
pipeline. The engine would need to detect when a `codec_id` resolves to a
pipeline definition rather than a leaf codec, and recurse into execution. This
is not yet implemented. Whether it is needed in practice — vs handling
composition at the plan level — is an open question.

---

## Codec Inventory

### Entries requiring verification

Several codec inventory entries are flagged with ⚠️ indicating lower confidence.
These should be verified against primary sources:

- `lzo`: exact variant and framing used in ORC; GPL v2 licensing implications
- `szip`: losslessness guarantee for float data
- `jpeg-xl`: TIFF tag number (50002)
- `jpeg-xr`: TIFF tag number (34892); licensing
- `tiff-predictor-3`: exact byte shuffle and delta order; row-by-row vs full array
- `fsst`: DuckDB identifier; framing conventions for symbol table storage
- `rle` (numcodecs): verify numcodecs exposes a general RLE codec
- `grib-grid-packing`, `grib-ccsds`, `grib-jpeg2000`: all three are low confidence

### Format-specific algorithm variants

TIFF's LZW is incompatible with standard LZW (`lzw-tiff` vs `lzw`). This
pattern — a format using a variant of a standard algorithm that is not
byte-compatible with the generic implementation — may recur when expanding to
new formats. Each new format integration should be checked for such variants.

### Lance codec compatibility

Verify whether Lance's RLE, dictionary, and delta encodings are byte-compatible
with the Parquet/ORC variants, or whether they require separate codec entries.

### Awareness classification edge cases

`tiff-predictor-2` and `png-filter` were classified as structure-aware (rather
than stride-aware) because they require `row_width` — dimensional layout
knowledge. `delta` was classified as type-aware (rather than stride-aware)
because it requires `dtype` for overflow semantics. These reclassifications
should be verified.

### ✅ Resolved inventory items

- deflate framing variants — handled by underlying zlib library
- netcdf-classic-record — no separate entry needed
- encode-only flag — implemented as `encode_only` in the original model;
  superseded by explicit bidirectional signatures
- pipeline direction and inversion — superseded by explicit bidirectional
  pipeline definitions
- ZFP / SZ lossy compressors — added as `zfp`, `sz2`, `sz3`
- Zarr v3 sharding — excluded (storage/access layer concern)

---

## Runtime

### Reference runtime

The proof-of-concept (chonkle) is a Python-hosted research implementation.
The reference runtime will be a native (Rust or C) implementation, which
eliminates the Canonical ABI performance bottleneck measured in the PoC. The
reference runtime is planned but not yet started.

### Native codec plugin discovery

Native codecs beyond the bundled standard library should be installable as
plugins and discoverable by the runtime at load time. The mechanism — system
packages, a plugin directory convention, a registry protocol, or some
combination — is not yet designed. The mechanism needs to support multiple
installed implementations for the same `codec_id` (e.g. CPU vs GPU variants,
SIMD-optimized vs reference implementations).

### Codec resolution precedence

The order in which the runtime checks for codec implementations — bundled
native codecs, installed native plugins, locally cached Wasm modules, remote
Wasm registries, pipeline `sources` hints — needs to be specified. The
proof-of-concept implementation has a working precedence chain that can inform
the spec, but the spec-level resolution order may differ.

---

## Distribution

### Signature verification scheme

Wasm codec artifacts may carry cryptographic signatures for integrity
verification. The key management model, trust roots, and signature format are
not yet specified.

### Embedded codec resolution

A codec URI could reference a byte range within a data file, enabling codecs
to travel with the data (similar to F3's embedded decoders). The URI format,
byte-range specification, and interaction with caching need design. See
[Distribution & Registry](04_runtime/02_distribution.md) and
[Comparison with F3](05_design/03_f3_comparison.md).

### warg / wa.dev maturity

Migration of Component Model codecs to the Bytecode Alliance's warg registry
is deferred until tooling matures. Core Wasm codecs are ineligible for warg
regardless and need a separate distribution channel.

---

## Specification Format

### WIT interface evolution

The current WIT interface (`chonkle:codec/transform@0.1.0`) requires both
`encode` and `decode` to be exported, even for unidirectional codecs (which
return an error for the unsupported direction). A future WIT revision could
use optional exports or separate worlds for unidirectional codecs. This is
blocked on WIT's support for optional exports.

### Codec inventory signature format migration

The codec inventory currently uses the original single-direction signature
format. It needs to be migrated to the new bidirectional format with explicit
`encode` and `decode` blocks. This is a mechanical transformation for symmetric
codecs (duplicate the single signature into both blocks) but requires careful
review for asymmetric codecs (`plain-dictionary`, `parquet-page-v2-split`,
`arrow-ipc-buffer`, `orc-string-direct`, `orc-string-dictionary`).

---

## Forward-Looking

### Format drivers and data orchestration

The [RFC on Format Drivers and Data Orchestration](07_proposals/01_orchestration.md)
proposes a plan format and execution model that sits above the codec layer —
adding storage I/O, format-specific callbacks, concurrency, and memory
management. This RFC is a draft; its relationship to the codec pipeline model
(particularly around conditional steps and the `choice` node) needs to be
resolved.

### CCRP

The Coalesced Chunk Retrieval Protocol (CCRP) is a conceptual data access
protocol that would use the Cylf codec layer as foundational infrastructure.
CCRP envisions querying multidimensional datasets stored in distributed block
stores, with responses including decoded chunks independent of source format.
An RFC exploring this direction is planned but not yet written.

### Chunk metadata model

As Cylf moves up the stack toward data orchestration (via the plan format) and
multi-chunk access (via CCRP), a richer metadata model for chunks may become
necessary — including chunk identity/coordinates, set-level metadata for shared
properties across chunks, and optional statistics for query-time filtering.
This is not part of the current codec layer spec. Whether it becomes part of
Cylf or remains the responsibility of format drivers and higher-level systems
is an open question.
