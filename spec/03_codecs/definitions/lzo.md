# lzo ⚠️

- **canonical-id**: `lzo`
- **aliases**: `{orc, "LZO"}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "lzo",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
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
- **spec**: https://www.oberhumer.com/opensource/lzo/ ⚠️
- **implementations**: [lzo](https://www.oberhumer.com/opensource/lzo/)
- **licensing**: GPL v2 ⚠️ — licensing may be a concern for some uses
- **notes**: ⚠️ Verify exact LZO variant and framing used in ORC. GPL license may be a concern.
