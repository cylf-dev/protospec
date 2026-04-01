# Motivation

## The Duplication Problem

Every chunked data format maintains its own codec ecosystem. Zarr has codecs in
zarr-python, zarrs (Rust), and the numcodecs library. HDF5 has its filter
pipeline. Parquet has its encoding layer. TIFF delegates to libtiff's
compression handlers. ORC, Arrow IPC, Lance, NetCDF, GRIB: each has its own
codec integration, its own build infrastructure, and its own library of
implementations.

The codecs themselves are largely the same. zstd, lz4, deflate, shuffle, delta,
bit-packing... The same algorithms appear across many formats, because they
solve the same fundamental problems: reduce data size, improve compressibility
through byte reordering, exploit structure in typed arrays. But each format's
library implements or wraps these algorithms independently, often in different
languages, with different APIs, different parameter conventions, and different
composition models.

This means the same work is done many times over. A new compression algorithm
must be adopted format by format. A bug found in one format's shuffle
implementation may exist in another's, unfixed, because nobody realizes they
share the same logic in different codebases. Even within a single format,
codecs are rebuilt across language ecosystems — Zarr's shuffle codec exists
independently in Python, Rust, and JavaScript, each implemented and maintained
separately.

The cost scales with the number of formats, the number of codecs, and the
number of languages. As all three grow—and all three are growing—the
maintenance burden and the fragmentation increase.

## What a Shared Foundation Enables

If codecs were implemented once, behind a common interface, every format could
use them. A bug fix, a performance optimization, or a new algorithm would
propagate to every format that builds on the shared layer, without per-format
integration work.

This is what Cylf provides: a shared codec layer that is format-agnostic. Cylf
defines how codecs describe their interfaces (typed signatures), how they
compose into multi-step processing pipelines (the DAG-based pipeline model), and
how they execute (a pipeline engine with a standard library of native codecs and
a WASM runtime for extensibility).

A format library that uses Cylf does not need to implement zstd, or shuffle, or
any other standard codec. It reads its own metadata, constructs a pipeline
definition that describes the decoding steps for a given chunk, and hands it to
the Cylf pipeline engine. The engine resolves the codecs, validates the
pipeline against codec signatures, and executes it. The format library focuses
on what only it knows—its own metadata structures, chunk layout, and access
patterns—and delegates the codec work to shared infrastructure. Formats that
require custom or non-standard codecs can implement and distribute them as
WASM modules, making them immediately available to users.

## Composition, Not Just Compression

Many formats apply codecs in sequence: a TIFF image tile might be differenced
(predictor), then compressed (zlib). A Zarr chunk might be shuffled, then
zstd-compressed. A Parquet page might be delta-encoded, then bit-packed, then
compressed, with the page itself split into separate level and value streams
that each take different codec paths.

These are not always simple linear chains. Parquet's page decoding involves
fan-out (one byte stream splitting into three), per-stream codec application,
and dictionary lookups that merge data from separate pages. Expressing this
naturally requires a DAG, not a stack.

Cylf's pipeline model supports arbitrary DAG composition: fan-out, fan-in, named
port routing, and explicit encode and decode pipelines. This means format drivers
can express their actual codec logic, however complex, as pipeline definitions,
rather than implementing custom orchestration code per format.

## Extensibility Without Fragmentation

New codecs and encoding strategies appear regularly. JPEG XL, LERC, ZFP, SZ3,
FSST are all examples of relatively recent additions to the ecosystem, each
currently tied to specific format libraries. When a data producer adopts a new
codec, every consumer needs a library update before it can read the data.

Cylf addresses this through its WASM runtime. Codecs compiled to WebAssembly
are portable (one binary, any platform), sandboxed (a codec cannot access host
memory or the filesystem), and distributable (fetchable from registries on
demand). When a data producer uses a new WASM codec, consumers with a
Cylf-based toolchain can decode the data without any library update: the
runtime fetches and executes the codec module automatically.

This does not mean every codec must be WASM. Cylf's standard library provides
native compiled implementations of widely-used codecs for maximum performance.
WASM is the extensibility mechanism, and it is the escape hatch that ensures
the system can handle new codecs without requiring every consumer to update
their tooling.

## Relationship to Existing Work

Cylf builds on ideas from several existing projects:

**Kerchunk and VirtualiZarr** demonstrate that you can virtualize access to
data across formats by reprojecting metadata. Cylf extends this idea to the
codec layer: rather than virtualizing format metadata into a single format's
model, it provides a format-agnostic codec interface that any format can target
directly.

**The Zarr codec ecosystem** has been working toward shared codec infrastructure
within its own community: numcodecs, zarrs in Rust, and codec support directly
in zarr-python. This demonstrates both the demand for shared codecs and the
challenge: even within a single format's ecosystem, codec implementations are
rebuilt per language. Cylf generalizes this effort to be format-agnostic, so the
same codec implementations serve Zarr, TIFF, Parquet, HDF5, and any other
format that builds on the layer. The pipeline model adds DAG composition and
bidirectional execution, which the current Zarr codec model does not provide.

**F3** (CMU SIGMOD 2025) proposes embedding WASM decoders directly in data files
for self-describing portability. Cylf shares F3's conviction that WASM is the
right substrate for portable codec execution, but takes a registry-first
approach to distribution: codecs are fetched from external registries rather
than embedded by default. Embedding codecs in files—via byte-range references
that resolve to codec modules within the data itself—is a viable resolution
strategy within Cylf's model and an open area of design. Beyond distribution,
Cylf's composition model is DAG-based rather than linear, and the interface is
formally typed rather than informal. See
[Comparison with F3](05_design/03_f3_comparison.md) for a detailed analysis.

## What Cylf Is Not

Cylf is a codec layer. It handles encoding and decoding of data chunks, the
transformation of bytes through pipelines of codecs. It does not define file
formats, manage storage, schedule I/O, or parse queries. Those responsibilities
belong to the format drivers, storage systems, and query engines that sit above
the codec layer.

Cylf defines the interface that format drivers target via codec signatures and
the pipeline schema, but it does not prescribe how format drivers are built or
how they interact with storage. The [Architecture](02_architecture.md) document
describes the boundaries in detail.

## Origin of the Name Cylf

Cylf is a stylized and search-optimized spelling of the word "sylph". Sylph is
both the genus of several South American hummingbird species and a word meaning
"air spirit". The inspiration for the name comes from the use of a hummingbird
in the logo of the related project [CCRP](https://ccrp.dev), the needs of which
also inspired the technical direction of this project.
