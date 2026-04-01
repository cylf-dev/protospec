# zigzag

- **canonical-id**: `zigzag`
- **aliases**: embedded inside ORC and Protocol Buffers encodings
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "zigzag",
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
- **spec**: https://protobuf.dev/programming-guides/encoding/#signed-ints
- **implementations**: Trivially implementable; included in protobuf, ORC, and Arrow libraries.
- **licensing**: Unencumbered
- **notes**: Maps signed integers to unsigned: 0→0, -1→1, 1→2, -2→3, etc. Improves subsequent varint or bitpack encoding. Used implicitly inside `orc-rle-v1`, `orc-rle-v2`, `orc-varint`.
