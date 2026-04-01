# delta-parquet

- **canonical-id**: `delta-parquet`
- **aliases**: `{parquet, "DELTA_BINARY_PACKED"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "delta-parquet",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "dtype": {
        "type": "string"
      },
      "block_size": {
        "type": "int",
        "default": 128
      },
      "miniblock_count": {
        "type": "int",
        "default": 4
      }
    },
    "outputs": {
      "bytes": {
        "type": "bytes"
      }
    }
  },
  "decode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "dtype": {
        "type": "string"
      }
    },
    "outputs": {
      "bytes": {
        "type": "bytes"
      }
    }
  }
}
```
- **conceptual decomposition**: `delta(dtype) → fixedscaleoffset(scale=1, offset=per_block_min) → bitpack(bit_width=per_miniblock)`. Not directly expressible as a pipeline because parameters vary per block within the stream.
- **spec**: https://parquet.apache.org/docs/file-format/data-pages/encodings/#delta-encoding-delta_binary_packed--5
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
