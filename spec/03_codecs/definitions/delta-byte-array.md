# delta-byte-array

- **canonical-id**: `delta-byte-array`
- **aliases**: `{parquet, "DELTA_BYTE_ARRAY"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "delta-byte-array",
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
- **conceptual decomposition**: `front-coding-split() → { delta-parquet(prefix_lengths), identity(suffixes) }`. Fan-out into two streams with delta on prefix lengths.
- **spec**: https://parquet.apache.org/docs/file-format/data-pages/encodings/#incremental-encoding-for-byte-arrays-aka-front-coding-delta_byte_array--7
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
- **notes**: Front coding: encodes shared prefixes between successive byte arrays.
