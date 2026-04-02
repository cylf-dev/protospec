# rle

- **canonical-id**: `rle`
- **aliases**: `{numcodecs, "numcodecs.RunLengthEncoding"}` ⚠️
- **awareness**: stride-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "rle",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "element_size": {
        "type": "uint"
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
      "element_size": {
        "type": "uint"
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
- **spec**: https://numcodecs.readthedocs.io/en/stable/ ⚠️ verify
- **implementations**: [numcodecs](https://github.com/zarr-developers/numcodecs) ⚠️
- **licensing**: MIT
- **notes**: ⚠️ Verify numcodecs exposes a general RLE codec.
