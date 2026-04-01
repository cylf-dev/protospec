# orc-rle-v1

- **canonical-id**: `orc-rle-v1`
- **aliases**: `{orc-v0, "RLE_V1"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "orc-rle-v1",
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
- **spec**: https://orc.apache.org/specification/ORCv0/
- **implementations**: [orc (Java)](https://github.com/apache/orc), [arrow (C++)](https://github.com/apache/arrow)
- **licensing**: Apache 2.0
- **notes**: Per-block adaptive selection between Run (fixed-delta) and Literals (varint). Signed integers use zigzag encoding internally. Not decomposable into a flat pipeline due to per-block strategy selection.
