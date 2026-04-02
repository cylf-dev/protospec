# astype

- **canonical-id**: `astype`
- **aliases**: `{numcodecs, "numcodecs.AsType"}`
- **awareness**: type-aware
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "astype",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "encode_dtype": {
        "type": "string"
      },
      "decode_dtype": {
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
      "encode_dtype": {
        "type": "string"
      },
      "decode_dtype": {
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
- **spec**: https://numcodecs.readthedocs.io/en/stable/astype.html
- **implementations**: [numcodecs](https://github.com/zarr-developers/numcodecs)
- **licensing**: MIT
- **notes**: Lossless if encode_dtype can represent all values exactly. Lossy otherwise.
