# orc-string-dictionary

- **canonical-id**: `orc-string-dictionary`
- **aliases**: `{orc, "DICTIONARY"}`, `{orc, "DICTIONARY_V2"}` (string columns)
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "orc-string-dictionary",
  "encode": {
    "inputs": {
      "indices": {
        "type": "bytes"
      },
      "dictionary": {
        "type": "bytes"
      },
      "dtype": {
        "type": "string"
      }
    },
    "outputs": {
      "data": {
        "type": "bytes"
      },
      "lengths": {
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
      "dtype": {
        "type": "string"
      }
    },
    "outputs": {
      "data": {
        "type": "bytes"
      },
      "lengths": {
        "type": "bytes"
      }
    }
  }
}
```
- **spec**: https://orc.apache.org/specification/ORCv1/
- **implementations**: [orc (Java)](https://github.com/apache/orc), [arrow (C++)](https://github.com/apache/arrow)
- **licensing**: Apache 2.0
- **notes**: Dictionary is per-stripe, supplied by format driver as a second `bytes` input. Multiple byte-stream inputs + multiple byte-stream outputs.
