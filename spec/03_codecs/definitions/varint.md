# varint

- **canonical-id**: `varint`
- **aliases**: `{protobuf, "varint"}`, `{orc, "base-128-varint"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "varint",
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
- **spec**: https://protobuf.dev/programming-guides/encoding/#varints
- **implementations**: Included in all protobuf implementations; trivially implementable.
- **licensing**: Unencumbered
- **notes**: Base-128 variable-length integer encoding. 7 data bits + 1 continuation bit per byte. Handles any unsigned integer type — small values use fewer bytes regardless of declared type width. Signed types should be zigzag-encoded first. See `orc-varint` pipeline decomposition.
