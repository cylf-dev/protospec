# F3 vs. Cylf: Comparison and Trade-off Analysis

References:
- [F3: The Open-Source Data File Format for the Future (CMU SIGMOD 2025)](https://db.cs.cmu.edu/papers/2025/zeng-sigmod2025.pdf)
- [github.com/future-file-format/F3](https://github.com/future-file-format/F3)

---

## What F3 proposes

F3 is a replacement for Parquet/ORC — a self-describing columnar file format
for analytics workloads.

- **IOUnit/EncUnit hierarchy**: IOUnit is the I/O parallelism unit (multiple
  pages); EncUnit is an atomic byte buffer encoded by a single algorithm.
  Stacking EncUnits within a column composes encodings hierarchically.
- **Wasm embedded in the data file**: When a new encoding is used, its Wasm
  decoder is embedded inside the file itself so any reader can decode it
  without pre-installed libraries. The codec travels with the data.
- **Vortex encodings**: 15+ built-in columnar encodings (bit-packing, ALP,
  dictionary, etc.) composable as linear chains per column.
- **FlatBuffers** for zero-copy metadata deserialization, replacing Parquet's
  Thrift.
- No explicit WIT or Component Model usage — a "general-purpose API" for
  Wasm decoders without a formal interface type system.

---

## Key differences

| Dimension | F3 | Cylf |
|---|---|---|
| **Scope** | Self-contained columnar file format | Format-agnostic codec layer used by any format |
| **Domain** | Structured tabular analytics (Parquet replacement) | Any chunked data — columnar formats (Parquet, ORC) and raster imagery (COG, satellite) |
| **Data model** | Typed columnar schema, FlatBuffer metadata | Opaque named byte buffers (port-map), schema-agnostic |
| **Codec composition** | Linear stack within a column (EncUnit chain) | Arbitrary DAG with named ports, fan-out/fan-in |
| **Codec distribution** | Embedded inside the data file | Fetched from external registries; embedded-in-file is an open design area |
| **Interface contract** | Informal "general-purpose API" — no WIT types | Formal WIT interface with explicit bidirectional signatures |
| **Bidirectionality** | Separate write (encode) and read (decode) paths | Explicit encode and decode definitions per codec and pipeline |
| **Self-description** | Data files are self-describing (decoder embedded) | Pipelines are separate from data; self-description via embedded codecs is an open area |
| **Metadata format** | FlatBuffers for column schema and statistics | Codec signatures embedded in `.wasm` binaries |

---

## F3's advantages over Cylf's approach

**Self-describing data is a strong archival guarantee.** If data is encoded with
a Cylf pipeline today, the pipeline definition is the only thing that knows how
to decode it. Lose the pipeline, lose the data. F3's "decoder travels with the
data" property is directly relevant for data that must be readable decades from
now without external dependencies. Cylf's codec resolution model could support
embedded codecs via byte-range references into the data file itself — this is an
open design area, not yet specified.

**Schema awareness enables query pushdown.** F3 knows column types, shapes, and
statistics. It can skip I/O for irrelevant columns, push down predicates, and
support partial decoding without reading the whole column. Cylf treats everything
as opaque bytes — any schema understanding is the responsibility of the format
driver sitting above the codec layer.

**Random access within EncUnits.** Because F3 tracks structure, it can decode a
row range without touching the rest of the column. Cylf pipelines process
complete chunks: a step receives a full port-map and emits a full port-map.

**Ecosystem integration target.** F3 aims to plug into Spark, DuckDB, and Pandas
by replacing Parquet. Cylf is infrastructure for format libraries, not a
query-engine integration point.

---

## Cylf's advantages over F3's approach

**Arbitrary DAG composition.** F3's EncUnit chain is linear per column. Cylf can
express fan-out (one output routed to multiple inputs), fan-in (multiple outputs
merged), and cross-step data routing via named wiring references. This is the
right model for both columnar formats (e.g., splitting a Parquet page into rep
levels, def levels, and values for independent processing) and multi-channel
imagery where byte planes are split, processed, and recombined — something a
linear stacking model cannot express without significant contortion.

**Formal interface contract.** Cylf codecs carry typed signatures with explicit
`encode` and `decode` blocks, each declaring their inputs and outputs. Wasm
codecs embed these signatures in the binary. The pipeline engine validates
signatures against pipeline requirements at load time. F3's "general-purpose
API" is informal by comparison — nothing prevents a codec from violating its
contract until runtime.

**Format-agnostic.** F3 is a file format — it defines how data is stored, not
just how it is encoded. Cylf is a codec layer that any format can build on.
A TIFF driver, a Zarr driver, and a Parquet driver can all use the same Cylf
codecs and pipeline engine. F3's codecs are tied to F3's format.

**Codec sandboxing with registry-based distribution.** Each Cylf Wasm codec runs
in an isolated sandbox. Codecs are fetched from controlled registries with
signature verification. F3 embeds Wasm from potentially untrusted data files,
which is a meaningful attack surface. F3 acknowledges a 10–30% performance
penalty for its sandboxed path.

**Concrete distribution model.** Cylf resolves codecs from `file://`, `https://`,
and `oci://` URIs, with remote artifacts cached locally. F3 notes "central
repository verification proposed" without specifying it.

---

## Where the problems diverge

F3 is optimizing for **reading existing data efficiently and portably** — a file
format problem. Cylf is optimizing for **composing transforms flexibly across
formats** — a shared infrastructure problem. These are different problem shapes,
and the approaches are not competing so much as operating at different levels of
the stack.

F3 could in principle use Cylf as its codec layer — replacing its informal
Wasm decoder API with Cylf's typed signatures, DAG pipelines, and registry-based
distribution. Cylf could in principle adopt F3's self-description approach —
embedding codec references (as byte-range URIs) within data files so that
the pipeline definition travels with the data. Neither project currently
targets this convergence, but the architectures are not incompatible.

**The gap Cylf does not currently address:** if data is encoded with a Cylf
pipeline, how does a reader know which pipeline to use to decode it? Currently
the answer is "the caller knows," which is appropriate for a tightly controlled
system but fragile at archival or interoperability scale. A content-addressed
pipeline identifier — resolvable to a pipeline definition in a registry — could
provide self-description at pipeline granularity rather than individual codec
granularity.