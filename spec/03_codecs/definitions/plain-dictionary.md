# plain-dictionary

- **canonical-id**: `plain-dictionary`
- **aliases**: `{parquet, "PLAIN_DICTIONARY"}`, `{parquet, "RLE_DICTIONARY"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "plain-dictionary",
  "encode": {
    "inputs": {
      "indices": {
        "type": "bytes"
      },
      "dictionary": {
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
      "indices": {
        "type": "bytes"
      },
      "dictionary": {
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
- **spec**: https://parquet.apache.org/docs/file-format/data-pages/encodings/#dictionary-encoding-plain_dictionary--2-and-rle_dictionary--8
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
- **notes**: `dictionary` is a second `bytes` input supplied by the format driver from a separate dictionary page. `PLAIN_DICTIONARY` (deprecated) and `RLE_DICTIONARY` are functionally equivalent.
