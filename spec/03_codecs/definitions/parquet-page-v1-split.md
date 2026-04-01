# parquet-page-v1-split

- **canonical-id**: `parquet-page-v1-split`
- **aliases**: `{parquet, "data-page-v1"}`
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "parquet-page-v1-split",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      }
    },
    "outputs": {
      "rep_bytes": {
        "type": "bytes"
      },
      "def_bytes": {
        "type": "bytes"
      },
      "value_bytes": {
        "type": "bytes"
      }
    }
  },
  "decode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      }
    },
    "outputs": {
      "rep_bytes": {
        "type": "bytes"
      },
      "def_bytes": {
        "type": "bytes"
      },
      "value_bytes": {
        "type": "bytes"
      }
    }
  }
}
```
- **spec**: https://parquet.apache.org/docs/file-format/data-pages/
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
- **notes**: Self-delimiting — decoder parses through RLE-framed level sections to find boundaries. No external length metadata. Fragile; prefer v2.
