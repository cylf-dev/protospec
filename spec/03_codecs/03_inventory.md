# Codec Inventory

Codec inventory for chunked array and columnar data formats. Each codec is a
named, versioned transformation with typed inputs and outputs. Codecs compose
into pipelines; a pipeline is itself a codec.

Entries flagged ⚠️ have lower confidence and should be verified against primary
sources before finalizing.

## Catalog Fields

Each codec entry includes:

- **canonical-id**: Primary key slug (will become a versioned URI)
- **aliases**: `{format, identifier}` pairs
- **awareness**: `pure-bytes` | `stride-aware` | `type-aware` | `structure-aware`
- **lossiness**: `lossless` | `lossy` | `conditional`
- **signature**: JSON codec signature with `encode` and `decode` blocks
  (see [Signature Model](01_signature.md))
- **pipeline**: (where applicable) pipeline decomposition into generic codecs
  (see [Pipeline Model](02_pipeline.md))
- **conceptual decomposition**: (where applicable) decomposition not directly
  expressible as a pipeline
- **spec**: Link(s) to canonical specification
- **implementations**: Known implementations
- **licensing**: Any restrictions
- **notes**: Caveats, open questions

## Pure-Bytes Codecs

Codec treats input as opaque bytes. No knowledge of element boundaries, data
types, or dimensional structure.

- [`zstd`](definitions/zstd.md)
- [`lz4`](definitions/lz4.md)
- [`deflate`](definitions/deflate.md)
- [`gzip`](definitions/gzip.md)
- [`bzip2`](definitions/bzip2.md)
- [`lzma`](definitions/lzma.md)
- [`snappy`](definitions/snappy.md)
- [`brotli`](definitions/brotli.md)
- [`lzo`](definitions/lzo.md) ⚠️
- [`lzw`](definitions/lzw.md)
- [`lzw-tiff`](definitions/lzw-tiff.md)
- [`packbits`](definitions/packbits.md)
- [`szip`](definitions/szip.md) ⚠️
- [`blosc`](definitions/blosc.md)
- [`jpeg-xl`](definitions/jpeg-xl.md)
- [`jpeg`](definitions/jpeg.md)
- [`jpeg2000`](definitions/jpeg2000.md)
- [`webp`](definitions/webp.md)
- [`jpeg-xr`](definitions/jpeg-xr.md) ⚠️
- [`no-compression`](definitions/no-compression.md)
- [`fletcher32`](definitions/fletcher32.md)
- [`crc32c`](definitions/crc32c.md)
- [`crc32`](definitions/crc32.md)
- [`orc-boolean-rle`](definitions/orc-boolean-rle.md)

## Stride-Aware Codecs

Codec needs to know element width (byte stride or bit width) to operate on
element boundaries, but does not need to know the semantic type.

- [`shuffle`](definitions/shuffle.md)
- [`bitshuffle`](definitions/bitshuffle.md)
- [`byte-stream-split`](definitions/byte-stream-split.md)
- [`rle-parquet`](definitions/rle-parquet.md)
- [`rle`](definitions/rle.md)

## Type-Aware Codecs

Codec needs to know the semantic data type (signed vs unsigned, float vs int,
physical type) to correctly encode or decode values.

- [`bitpack`](definitions/bitpack.md)
- [`fsst`](definitions/fsst.md)
- [`nbit`](definitions/nbit.md)
- [`scale-offset`](definitions/scale-offset.md)
- [`lerc`](definitions/lerc.md)
- [`delta-parquet`](definitions/delta-parquet.md)
- [`delta-length-byte-array`](definitions/delta-length-byte-array.md)
- [`delta-byte-array`](definitions/delta-byte-array.md)
- [`delta`](definitions/delta.md)
- [`fixedscaleoffset`](definitions/fixedscaleoffset.md)
- [`quantize`](definitions/quantize.md)
- [`bitround`](definitions/bitround.md)
- [`astype`](definitions/astype.md)
- [`bytes`](definitions/bytes.md)
- [`plain-parquet`](definitions/plain-parquet.md)
- [`plain-dictionary`](definitions/plain-dictionary.md)
- [`orc-rle-v1`](definitions/orc-rle-v1.md)
- [`orc-rle-v2`](definitions/orc-rle-v2.md)
- [`orc-varint`](definitions/orc-varint.md)
- [`orc-string-direct`](definitions/orc-string-direct.md)
- [`orc-string-dictionary`](definitions/orc-string-dictionary.md)
- [`vlen-utf8`](definitions/vlen-utf8.md)
- [`vlen-bytes`](definitions/vlen-bytes.md)
- [`categorize`](definitions/categorize.md)
- [`zigzag`](definitions/zigzag.md)
- [`varint`](definitions/varint.md)
- [`zfp`](definitions/zfp.md)
- [`sz2`](definitions/sz2.md) ⚠️
- [`sz3`](definitions/sz3.md) ⚠️

## Structure-Aware Codecs

Codec needs format-specific structural knowledge — dimensional layout, page
structure, image geometry, or format-specific metadata beyond simple type info.

- [`transpose`](definitions/transpose.md)
- [`tiff-predictor-2`](definitions/tiff-predictor-2.md)
- [`tiff-predictor-3`](definitions/tiff-predictor-3.md)
- [`png-filter`](definitions/png-filter.md)
- [`parquet-page-v1-split`](definitions/parquet-page-v1-split.md)
- [`parquet-page-v2-split`](definitions/parquet-page-v2-split.md)
- [`arrow-ipc-buffer`](definitions/arrow-ipc-buffer.md)
- [`tiff-old-jpeg`](definitions/tiff-old-jpeg.md)
- [`ccitt-group3`](definitions/ccitt-group3.md)
- [`ccitt-group4`](definitions/ccitt-group4.md)
- [`grib-grid-packing`](definitions/grib-grid-packing.md) ⚠️
- [`grib-ccsds`](definitions/grib-ccsds.md) ⚠️
- [`grib-jpeg2000`](definitions/grib-jpeg2000.md) ⚠️

## Explicitly Excluded

The following appear in codec registries but are excluded from this catalog for
the reasons given.

### Serialization bridges

These are object serialization formats, not typed array codecs.

#### json (excluded)

- **identifier**: `numcodecs.JSON`
- **reason**: Encodes arbitrary Python objects to JSON bytes. Decode output is
  runtime-dependent, not a typed numeric array.

#### msgpack (excluded)

- **identifier**: `numcodecs.MsgPack`
- **reason**: Binary serialization for arbitrary object graphs. Not a typed
  array codec.

#### pickle (excluded)

- **identifier**: `numcodecs.Pickle`
- **reason**: Python-specific. Security concerns (arbitrary code execution on
  unpickle). Excluded on portability and security grounds.

### Storage/access layer operations

These are registered as codecs in their respective formats but do not perform
data transformation in the sense used by this catalog. They reorganize how
chunks are stored and accessed — a concern of the storage/access layer above
the codec layer.

#### sharding-indexed (excluded)

- **identifier**: `{zarr-v3, "sharding_indexed"}`
- **spec**:
  https://zarr-specs.readthedocs.io/en/latest/v3/codecs/sharding-indexed/index.html
- **reason**: Sharding splits a chunk ("shard") into inner sub-chunks, applies
  a nested codec pipeline to each sub-chunk independently, concatenates the
  compressed sub-chunks, and appends an index (offset + length per sub-chunk)
  with its own codec pipeline. This is a storage organization operation — it
  determines how data is laid out within a storage object for efficient partial
  access — not a data transformation. No byte of actual array data is
  transformed by sharding itself; the transformation is delegated to the inner
  codec pipelines, which are composed of codecs from this catalog.

  Zarr v3 registers sharding as a codec (specifically an `array → bytes` codec)
  because Zarr's codec pipeline is the only extension point available for
  operations between the array and storage layers. This is an overloading of the
  codec concept. A more precise model would place sharding in a separate
  storage/access layer that sits above the codec layer — responsible for
  chunking strategy, sub-chunk indexing, and partial I/O — while the codec layer
  handles the actual data transformations applied to each chunk or sub-chunk.

  The storage/access layer could potentially be modeled with a similar signature
  and pipeline structure to what this catalog uses for codecs, but that design
  is out of scope here.

## Open Questions and Issues

1. ~~**deflate framing variants (resolved)**~~: Handled by underlying zlib
   library.
2. **lzo licensing**: GPL v2 may be problematic. Worth flagging.
3. **szip losslessness for floats**: Verify whether SZIP guarantees
   losslessness for float data.
4. **TIFF predictor-3 traversal order**: Verify row-by-row vs full flattened
   array.
5. **ORC v2 codec evolution**: ORC v2 proposes removing RLEv1/v2. Monitor.
6. **GRIB entries**: All three are low confidence.
7. ~~**netcdf-classic-record (resolved)**~~: No separate entry needed.
8. **variable-width-split pattern**: `orc-string-direct`, `arrow-ipc-buffer`,
   and `delta-length-byte-array` share a lengths/values split pattern. A
   generic `variable-width-split` codec may unify these, but input framing
   varies across formats. TBD.
9. **Data-derived inputs**: Should data-derived inputs (built during encode,
   required for decode, stored in metadata — e.g. FSST `symbol_table`) have an
   explicit flag in the signature?
10. ~~**Encode-only flag (resolved)**~~: Resolved by the `encode`/`decode`
    block structure in signatures. Direction-specific parameters appear only in
    the relevant direction's `inputs`. See [Signature Model](01_signature.md).
11. **Format-specific algorithm variants**: `lzw-tiff` pattern. Check when
    expanding to new formats.
12. **Checksum codec scope**: `fletcher32`, `crc32c`, and `crc32` are now
    included. Determine whether additional checksum variants exist in target
    formats. Adler-32 (used in zlib framing) is embedded in the deflate codec's
    framing, not a separate codec.
13. ~~**Pipeline direction and inversion (resolved)**~~: Resolved by explicit
    `encode` and `decode` blocks in pipeline definitions. Each direction's DAG
    is defined independently. See [Pipeline Model](02_pipeline.md).
14. **Conditional pipeline steps**: `orc-varint` zigzag step is conditional on
    signedness. How to model conditional steps is TBD.
15. **Per-block parameter variation**: `delta-parquet` uses
    per-block/per-miniblock parameters. These vary within a single stream,
    preventing direct pipeline decomposition. Consider block-wise
    parameterization.
16. **TIFF bitpack alias**: Verify whether TIFF sub-byte sample packing is
    correctly modeled as `bitpack(bit_width=bits_per_sample)`.
17. **Lance codec compatibility**: Verify whether Lance's RLE, dictionary, and
    delta are byte-compatible with Parquet/ORC variants.
18. **Codec URI scheme**: Codec IDs will become versioned URIs. Scheme TBD.
19. ~~**ZFP / SZ lossy compressors (resolved)**~~: Added as `zfp`, `sz2`,
    `sz3`. All are atomic (not decomposable). Transform-based (ZFP) vs
    prediction-based (SZ) approaches to error-bounded lossy compression.
20. ~~**Zarr v3 sharding codec (resolved)**~~: Excluded. Sharding is a
    storage/access layer concern, not a data transformation. See the
    `sharding-indexed` entry in Explicitly Excluded.
21. **Self-describing codec boundary**: Blosc is atomic because its header is
    embedded in the chunk. Formalize the criterion.
22. **Awareness classification edge cases**: `tiff-predictor-2` and
    `png-filter` moved from stride-aware to structure-aware because they
    require `row_width` (dimensional layout knowledge). `delta` moved from
    stride-aware to type-aware because it requires `dtype` for overflow
    semantics. These reclassifications should be verified.
