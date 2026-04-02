# bzip2

- **canonical-id**: `bzip2`
- **aliases**: `{zarr-v2, "numcodecs.BZ2"}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "bzip2",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "level": {
        "type": "int",
        "default": 1
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
- **spec**: https://sourceware.org/bzip2/
- **implementations**: [bzip2](https://sourceware.org/bzip2/), [bzip2-rs](https://github.com/alexcrichton/bzip2-rs)
- **licensing**: BSD-like (bzip2 license)
