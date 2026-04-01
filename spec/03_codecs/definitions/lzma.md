# lzma

- **canonical-id**: `lzma`
- **aliases**: `{zarr-v2, "numcodecs.LZMA"}`, `{tiff, 34925}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "lzma",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "format": {
        "type": "int",
        "default": 1
      },
      "check": {
        "type": "int",
        "default": -1
      },
      "preset": {
        "type": "int",
        "required": false
      },
      "filters": {
        "type": "string",
        "required": false
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
- **spec**: https://tukaani.org/xz/xz-file-format.txt
- **implementations**: [xz-utils](https://tukaani.org/xz/), [lzma-rs](https://github.com/gendx/lzma-rs)
- **licensing**: Public domain / 0BSD
- **notes**: TIFF (tag 34925) uses LZMA2 via liblzma. The framing difference (XZ container vs raw LZMA2 stream) is handled transparently by liblzma depending on invocation context.
