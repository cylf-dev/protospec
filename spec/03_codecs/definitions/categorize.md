# categorize

- **canonical-id**: `categorize`
- **aliases**: `{numcodecs, "numcodecs.Categorize"}`
- **awareness**: type-aware
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "categorize",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "labels": {
        "type": "string"
      },
      "dtype": {
        "type": "string"
      },
      "astype": {
        "type": "string",
        "default": "u1"
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
      "labels": {
        "type": "string"
      },
      "dtype": {
        "type": "string"
      },
      "astype": {
        "type": "string",
        "default": "u1"
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
- **spec**: https://numcodecs.readthedocs.io/en/stable/categorize.html
- **implementations**: [numcodecs](https://github.com/zarr-developers/numcodecs)
- **licensing**: MIT
- **notes**: Maps categorical values to integer codes. Analogous to dictionary encoding. Currently Python-specific in framing. Lossless if all values in labels; error on unknown values.
