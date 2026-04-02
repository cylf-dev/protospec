# orc-string-direct

- **canonical-id**: `orc-string-direct`
- **aliases**: `{orc, "DIRECT"}` (string columns)
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "orc-string-direct",
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
      "lengths": {
        "type": "bytes"
      },
      "data": {
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
      "lengths": {
        "type": "bytes"
      },
      "data": {
        "type": "bytes"
      }
    }
  }
}
```
- **spec**: https://orc.apache.org/specification/ORCv1/
- **implementations**: [orc (Java)](https://github.com/apache/orc), [arrow (C++)](https://github.com/apache/arrow)
- **licensing**: Apache 2.0
- **notes**: Fan-out: produces two output streams. Lengths stream uses orc-rle-v1 or orc-rle-v2.
