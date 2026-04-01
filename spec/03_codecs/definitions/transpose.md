# transpose

- **canonical-id**: `transpose`
- **aliases**: `{zarr-v3, "transpose"}`
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "transpose",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "shape": {
        "type": "uint[]"
      },
      "order": {
        "type": "uint[]"
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
      "shape": {
        "type": "uint[]"
      },
      "order": {
        "type": "uint[]"
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
- **spec**: https://zarr-specs.readthedocs.io/en/latest/v3/codecs/transpose/v1.0.html
- **implementations**: [zarr-python](https://github.com/zarr-developers/zarr-python)
- **licensing**: MIT
- **notes**: Reorders axes before serialization. Equivalent to numpy transpose.
