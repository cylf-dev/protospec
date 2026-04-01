# Codec Signature Model

A codec signature is a typed description of a codec's interface: what inputs it
accepts and what outputs it produces when encoding, when decoding, or both.  A
codec may be bidirectional (implementing both `encode` and `decode`) or
unidirectional (implementing only one direction, as with lossy codecs whose
transformations are not reversible).

Every codec, whether a native standard library implementation or a WASM module,
carries its own signature. The pipeline engine discovers a codec's signature
when it loads the module and validates it against the pipeline's requirements
before execution begins.

Signatures are leveraged in the [Codec Inventory](03_inventory.md), which
catalogs known codecs with their signatures, format aliases, and composition
properties.

## Type Vocabulary

Signatures use a small, fixed type vocabulary:

| Type | Description |
|---|---|
| `bytes` | A byte stream |
| `uint` | Unsigned integer scalar |
| `int` | Signed integer scalar |
| `float` | Floating point scalar |
| `bool` | Boolean scalar |
| `string` | String value (e.g. dtype name, encoding mode) |
| `uint[]` | Array of unsigned integers (e.g. shape, permutation) |
| `dtype_desc` | Dtype descriptor (base type, width, signedness) |

All data flowing through codec pipelines is ultimately `bytes`. The non-bytes
types are used for parameters: codec configuration values like compression
level, element size, data type, and shape. In the port-map wire format,
non-bytes values are serialized as UTF-8 JSON strings.

## Signature Structure

A signature has two top-level blocks, `encode` and `decode`, each declaring
the input and output port mappings for that direction. A **port** is a named,
typed channel on a codec's interface. Ports typed `bytes` predominantly carry
the data being transformed, while ports with scalar types (`int`, `string`,
`bool`, `uint[]`, `dtype_desc`) are used for parameters to configure codec
behavior.

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

### `codec_id` (string, required)

The canonical identifier for this codec. Used in pipeline step definitions and
for codec resolution. Codec identifiers are short slugs (e.g. `zstd`, `shuffle`,
`tiff-predictor-2`). A versioned URI scheme for codec identifiers is planned but
not yet defined — see [Open Questions](../06_future.md).

### `encode` (object, optional)

The codec's interface when running in the encode direction.

### `decode` (object, optional)

The codec's interface when running in the decode direction.

At least one of `encode` or `decode` must be present. A codec that provides
only one direction is unidirectional; validation must reject attempts to use it
in the absent direction. Lossy codecs (e.g. `quantize`, `round`) may only define
`encode`, since the transformation is not reversible.

Each direction block contains:

### `inputs` (object, required)

A map of input port descriptors for this direction. Each key is a port name;
each value describes the port:

- `type` (string, required): one of the types in the vocabulary above.
- `required` (bool, optional, default `true`): whether the caller must supply
  this input. A `default` value implies `required: false`.
- `default` (optional): the default value when the input is not supplied.
  Specifying a `default` implicitly sets `required` to `false`; validation
  should fail if a default is set when `required` is `true`.

### `outputs` (object, required)

A map of output port descriptors for this direction. Each key is a port name;
each value describes the port:

- `type` (string, required): one of the types in the vocabulary above.

## Direction-Specific Parameters

Because each direction has its own `inputs` block, parameters that are only
relevant in one direction simply appear in that direction's inputs and not the
other's. The structure makes direction specificity explicit, no directional
annotations on parameters required.

In the "zstd" example above, `level` appears only in `encode.inputs` — it
configures compression and has no meaning during decompression. `dictionary`
appears in both directions because zstd can optionally use an external
dictionary for both encoding and decoding. A checksum codec like `crc32c` might
have `throw_error` only in `decode.inputs`:

```json
{
  "codec_id": "crc32c",
  "encode": {
    "inputs": {
      "bytes": {"type": "bytes"}
    },
    "outputs": {
      "bytes": {"type": "bytes"}
    }
  },
  "decode": {
    "inputs": {
      "bytes": {"type": "bytes"},
      "throw_error": {"type": "bool", "default": true}
    },
    "outputs": {
      "bytes": {"type": "bytes"}
    }
  }
}
```

## Awareness Taxonomy

Codecs are classified by **awareness** — the minimum knowledge a codec needs
about the structure and semantics of the data it transforms. Awareness
determines what data-descriptive information a format driver must extract from
format metadata and supply to the codec. A codec at any awareness level may
still accept additional parameters for algorithm configuration or side inputs
(e.g.  compression level, dictionary).

The idea of awareness is perhaps better understood by means of comparison.
Consider Zarr v3's codec classification, which distinguishes `ArrayArrayCodec`,
`ArrayBytesCodec`, and `BytesBytesCodec` based on whether the codec's interface
accepts or produces typed arrays versus byte streams. In the Cylf model, data
being transformed is always bytes, no distinct "array" type exists at the data
port level. What Zarr captures as a type-level distinction (array vs bytes) is
instead captured here as a degree of awareness: a codec that Zarr would
classify as an `ArrayArrayCodec` (e.g. `transpose`) is one that needs
structural knowledge about the byte stream—its shape, dtype, and element
layout—supplied via data-descriptive parameter ports. The data itself is still
bytes in and bytes out.

Awareness decouples codecs from any specific data model. Zarr's classification
assumes codecs operate on arrays, and the codec types themselves enforce that
assumption. By treating data uniformly as bytes and expressing structural
knowledge through parameter ports, the Cylf model is not limited to array data.
The same codec definitions and pipeline machinery can serve arrays, rasters,
point clouds, tabular formats, or any other structure: format drivers simply
need to supply whatever data-descriptive parameters a codec's awareness level
requires.

### Pure-bytes

The codec treats input as opaque bytes. No knowledge of element boundaries, data
types, or dimensional structure is needed — no data-descriptive parameter ports
are required.

Examples: `zstd`, `lz4`, `deflate`, `gzip`, `blosc`, `snappy`, `brotli`,
`jpeg`, `jpeg2000`, `webp`

### Stride-aware

The codec needs to know element width (byte stride or bit width) to operate on
element boundaries, but does not need the semantic type. Requires an
`element_size` or `bit_width` parameter in addition to `bytes`.

Examples: `shuffle`, `bitshuffle`, `byte-stream-split`, `rle-parquet`

### Type-aware

The codec needs the semantic data type (signed vs unsigned, float vs int,
physical type) to correctly encode or decode values. Requires a `dtype` or
equivalent parameter.

Examples: `delta`, `bitpack`, `zigzag`, `varint`, `fixedscaleoffset`,
`plain-parquet`, `plain-dictionary`, `zfp`, `sz3`

### Structure-aware

The codec needs format-specific structural knowledge — dimensional layout, page
structure, image geometry, or format-specific metadata beyond simple type info.
Requires parameters like `row_width`, `shape`, `image_width`, or structural
length fields.

Examples: `transpose`, `tiff-predictor-2`, `tiff-predictor-3`, `png-filter`,
`parquet-page-v2-split`, `arrow-ipc-buffer`

### Implications for pipeline construction

The awareness classification, which is implicitly encoded into a codec's
signature, tells a format driver what data-descriptive information it must
extract from format metadata and wire into the pipeline. A pipeline using only
pure-bytes codecs needs no data-descriptive parameters — the format driver only
needs to supply the byte stream itself. A pipeline with a structure-aware codec
like `tiff-predictor-2` requires the format driver to supply `element_size` and
`row_width` — values derived from the TIFF IFD's `BitsPerSample` and
`ImageWidth` tags.

## Codec Identification

Codec identifiers are currently short slugs: `zstd`, `shuffle`,
`tiff-predictor-2`. These are human-readable and unambiguous within the codec
inventory.

A versioned URI scheme is planned to support iteration on codec specifications
without breaking existing pipelines. The exact format is an open question — see
[Open Questions](../06_future.md).

## Design Notes

### Output port convention for type-dependent codecs

Some codecs like `arrow-ipc-buffer` produce a different number of meaningful
output buffers depending on runtime parameters (`arrow_type`, `nullable`). The
convention is that the signature declares all possible output ports statically,
and the codec always emits all of them — ports that are not meaningful for a
given invocation are emitted as zero-length byte buffers. This avoids the need
for variable-arity output mechanics in the signature model and keeps downstream
pipeline wiring static and predictable.

## Open Questions

- **Codec ID versioning and URIs.** Codec identifiers are currently unversioned
  slugs. A versioned URI scheme (e.g. `zstd@1.0.0`) would support iterating on
  codec specifications without breaking existing pipelines. The format and
  semantics of version identifiers are not yet defined. More broadly, should
  codec IDs be URIs? If so, should they be expected to be dereferenceable (i.e.
  fetchable to retrieve the codec specification), or are opaque URI identifiers
  sufficient?
- **Codec ID namespaces.** Current codec IDs are flat slugs with no namespace
  structure. Should codec IDs support namespacing (e.g. `zarr/shuffle`,
  `tiff/predictor-2`, `org.example/custom-codec`) to avoid collisions between
  independently-developed codecs and to group related codecs by origin or
  domain? If so, what is the namespace syntax, who governs namespace
  allocation, and how do namespaces interact with the URI and versioning
  schemes above?
- **Codec specification vs implementation metadata.** A signature describes a
  codec's *specification*, its interface contract, not a particular
  implementation. Multiple implementations (e.g. `zstd-c`, `zstd-rust`) may
  implement the same spec and therefore share the same signature. Currently
  Chonkle requires signatures to include an implementation field, but this
  conflates the spec with its implementation. Implementation metadata (language,
  library, version, performance characteristics) likely belongs in a separate
  object rather than in the signature itself. A deeper question is whether
  implementations should advertise their own signature at all, or whether the
  signature should be authoritative from the spec side, with implementations
  simply declaring which codec spec they implement. For non-canonical or custom
  codecs without a published spec, how does a consumer discover the signature?
- **JSON Schema adoption.** The signature format could be formalized using
  JSON Schema rather than a bespoke schema description, which would enable
  standard validation tooling and editor support.

## Further Reading

- [Codec Inventory](03_inventory.md) — catalog of codecs with signatures,
  aliases, and awareness classifications
- [Pipeline Model](02_pipeline.md) — how codecs compose into executable DAGs
- [Open Questions](../06_future.md) — consolidated list of unresolved design
  questions across the Cylf ecosystem
