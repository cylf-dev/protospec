# orc-rle-v2

- **canonical-id**: `orc-rle-v2`
- **aliases**: `{orc-v1, "RLE_V2"}`, `{orc-v2, "RLE_V2"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "orc-rle-v2",
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
- **spec**: https://orc.apache.org/specification/ORCv1/
- **implementations**: [orc (Java)](https://github.com/apache/orc), [arrow (C++)](https://github.com/apache/arrow)
- **licensing**: Apache 2.0
- **notes**: Per-block adaptive selection from four sub-encodings via 2-bit header: Short Repeat, Direct (bitpacked), Patched Base, Delta. Not decomposable. Signed integers use zigzag internally.
