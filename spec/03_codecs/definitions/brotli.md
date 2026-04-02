# brotli

- **canonical-id**: `brotli`
- **aliases**: `{parquet, "BROTLI"}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "brotli",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "quality": {
        "type": "int",
        "default": 11
      },
      "lgwin": {
        "type": "int",
        "default": 22
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
- **spec**: https://www.rfc-editor.org/rfc/rfc7932
- **implementations**: [brotli (C reference)](https://github.com/google/brotli)
- **licensing**: MIT
