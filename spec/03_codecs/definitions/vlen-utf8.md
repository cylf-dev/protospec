# vlen-utf8

- **canonical-id**: `vlen-utf8`
- **aliases**: `{numcodecs, "numcodecs.VLenUTF8"}`, `{zarr-v2, "numcodecs.VLenUTF8"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "vlen-utf8",
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
- **spec**: https://numcodecs.readthedocs.io/en/stable/vlen.html ⚠️
- **implementations**: [numcodecs](https://github.com/zarr-developers/numcodecs)
- **licensing**: MIT
- **notes**: Serializes variable-length UTF-8 strings into a flat bytestream. Output length not predictable from dtype/shape. Breaks fixed-stride assumption. Currently Zarr/Python-specific framing.
