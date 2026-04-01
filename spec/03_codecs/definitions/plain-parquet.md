# plain-parquet

- **canonical-id**: `plain-parquet`
- **aliases**: `{parquet, "PLAIN"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "plain-parquet",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "physical_type": {
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
      "physical_type": {
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
- **spec**: https://parquet.apache.org/docs/file-format/data-pages/encodings/#plain-plain--0
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
- **notes**: Encoding varies by physical type. Fixed-width: packed little-endian. BYTE_ARRAY: 4-byte length + data. BOOLEAN: 8 per byte LSB-first.
