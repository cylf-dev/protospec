# delta

- **canonical-id**: `delta`
- **aliases**: `{zarr-v3, "numcodecs.delta"}`, `{numcodecs, "numcodecs.Delta"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "delta",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "dtype": {
        "type": "string"
      },
      "element_size": {
        "type": "uint"
      },
      "shape": {
        "type": "uint[]",
        "required": false
      },
      "order": {
        "type": "string",
        "default": "C"
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
      },
      "element_size": {
        "type": "uint"
      },
      "shape": {
        "type": "uint[]",
        "required": false
      },
      "order": {
        "type": "string",
        "default": "C"
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
- **spec**: https://numcodecs.readthedocs.io/en/stable/delta.html
- **implementations**: [numcodecs](https://github.com/zarr-developers/numcodecs)
- **licensing**: MIT
- **notes**: Differences between successive elements. Unlike tiff-predictor-2, operates on the full array (not row-by-row) unless shape/order constrain traversal. `dtype` affects overflow behavior.
