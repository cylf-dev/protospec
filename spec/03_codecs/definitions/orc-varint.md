# orc-varint

- **canonical-id**: `orc-varint`
- **aliases**: `{orc, "VARINT"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "orc-varint",
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
- **pipeline** (decode, signed types):
```json
{
  "codec_id": "orc-varint",
  "decode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "dtype": {
        "type": "string"
      }
    },
    "steps": {
      "decode_varint": {
        "codec_id": "varint",
        "inputs": {
          "bytes": "inputs.bytes"
        }
      },
      "unzigzag": {
        "codec_id": "zigzag",
        "inputs": {
          "bytes": "steps.decode_varint.bytes",
          "dtype": "inputs.dtype"
        }
      }
    },
    "outputs": {
      "bytes": "steps.unzigzag.bytes"
    }
  }
}
```
Note: For unsigned types, the zigzag step is skipped. Modeling of conditional steps is TBD.
- **spec**: https://orc.apache.org/specification/ORCv1/
- **implementations**: [orc (Java)](https://github.com/apache/orc)
- **licensing**: Apache 2.0
- **notes**: Base-128 variable-length integer encoding with zigzag for signed types. See generic `varint` and `zigzag` codecs.
