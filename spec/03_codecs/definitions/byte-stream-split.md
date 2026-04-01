# byte-stream-split

- **canonical-id**: `byte-stream-split`
- **aliases**: `{parquet, "BYTE_STREAM_SPLIT"}`
- **awareness**: stride-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "byte-stream-split",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "element_size": {
        "type": "uint"
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
      "element_size": {
        "type": "uint"
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
- **spec**: https://parquet.apache.org/docs/file-format/data-pages/encodings/#byte-stream-split-byte_stream_split--11
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
- **notes**: Identical in effect to HDF5 shuffle. Transposes byte matrix: all byte-0s contiguous, then all byte-1s, etc.
