# delta-length-byte-array

- **canonical-id**: `delta-length-byte-array`
- **aliases**: `{parquet, "DELTA_LENGTH_BYTE_ARRAY"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "delta-length-byte-array",
  "encode": {
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
- **conceptual decomposition**: `variable-width-split() → { delta-parquet(lengths), identity(values) }`. Fan-out, then per-stream codec.
- **spec**: https://parquet.apache.org/docs/file-format/data-pages/encodings/#delta-encoding-of-string-lengths-delta_length_byte_array--6
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
- **notes**: Stores lengths via delta-binary-packed encoding, followed by concatenated raw bytes.
