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

Signatures are also the basis for the [Codec Inventory](03_inventory.md),
which catalogs known codecs with their signatures, format aliases, and
composition properties.

---

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

---

## Signature Structure

A signature has two top-level blocks, `encode` and `decode`, each declaring
the inputs and outputs for that direction.

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
  Specifying a `default` implicitly sets `required` to `false`.

### `outputs` (object, required)

A map of output port descriptors for this direction. Each key is a port name;
each value describes the port:

- `type` (string, required): one of the types in the vocabulary above.

---

## Direction-Specific Parameters

Because each direction has its own `inputs` block, parameters that are only
relevant in one direction simply appear in that direction's inputs and not the
other's. No `encode_only` or `decode_only` flags are needed — the structure
makes direction specificity explicit.

In the zstd example above, `level` appears only in `encode.inputs` — it
configures compression and has no meaning during decompression. `dictionary`
appears in both directions because zstd can use a dictionary for both encoding
and decoding. A checksum codec like `crc32c` might have `throw_error` only in
`decode.inputs`:

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

The distinction between data ports and parameter ports is also structural: a
port that appears in `inputs` but not in `outputs` for a given direction is a
parameter. Parameters configure the operation but data does not flow through
them. `level` is in `encode.inputs` but not in `encode.outputs`; it's a
parameter. `bytes` is in both `encode.inputs` and `encode.outputs`; it's data.

---

## Awareness Taxonomy

Codecs are classified by **awareness** — the minimum knowledge a codec needs
about the data beyond raw bytes. Awareness determines what parameters a pipeline
must supply for a codec to execute correctly.

### Pure-bytes

The codec treats input as opaque bytes. No knowledge of element boundaries, data
types, or dimensional structure is needed. Only the `bytes` port is required.

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

The awareness classification tells a format driver what information it must
extract from format metadata and wire into the pipeline. A pipeline using only
pure-bytes codecs needs no parameters beyond the data itself. A pipeline with a
structure-aware codec like `tiff-predictor-2` requires the format driver to
supply `element_size` and `row_width` — values derived from the TIFF IFD's
`BitsPerSample` and `ImageWidth` tags.

---

## Codec Identification

Codec identifiers are currently short slugs: `zstd`, `shuffle`,
`tiff-predictor-2`. These are human-readable and unambiguous within the codec
inventory.

A versioned URI scheme is planned to support iteration on codec specifications
without breaking existing pipelines. The exact format is an open question — see
[Open Questions](../06_future.md).

---

## Design Notes

### Output port convention for type-dependent codecs

Some codecs like `arrow-ipc-buffer` produce a different number of meaningful
output buffers depending on runtime parameters (`arrow_type`, `nullable`). The
convention is that the signature declares all possible output ports statically,
and the codec always emits all of them — ports that are not meaningful for a
given invocation are emitted as zero-length byte buffers. This avoids the need
for variable-arity output mechanics in the signature model and keeps downstream
pipeline wiring static and predictable.

---

## Open Questions

- **Codec ID versioning.** Codec identifiers are currently unversioned slugs. A
  versioned URI scheme (e.g. `zstd@1.0.0`) would support iterating on codec
  specifications without breaking existing pipelines. The format and semantics of
  version identifiers are not yet defined.
- **JSON Schema adoption.** The signature format could be formalized using
  JSON Schema rather than a bespoke schema description, which would enable
  standard validation tooling and editor support.

---

## Further Reading

- [Codec Inventory](03_inventory.md) — catalog of codecs with signatures,
  aliases, and awareness classifications
- [Pipeline Model](02_pipeline.md) — how codecs compose into executable DAGs
- [Open Questions](../06_future.md) — consolidated list of unresolved design
  questions across the Cylf ecosystem
