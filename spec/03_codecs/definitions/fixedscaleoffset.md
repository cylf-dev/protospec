# fixedscaleoffset

- **canonical-id**: `fixedscaleoffset`
- **aliases**: `{numcodecs, "numcodecs.FixedScaleOffset"}`, `{zarr-v3, "numcodecs.fixedscaleoffset"}`
- **awareness**: type-aware
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "fixedscaleoffset",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "offset": {
        "type": "float"
      },
      "scale": {
        "type": "float"
      },
      "dtype": {
        "type": "string"
      },
      "astype": {
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
      },
      "offset": {
        "type": "float"
      },
      "scale": {
        "type": "float"
      },
      "dtype": {
        "type": "string"
      },
      "astype": {
        "type": "string",
        "required": false
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
- **spec**: https://numcodecs.readthedocs.io/en/stable/fixedscaleoffset.html
- **implementations**: [numcodecs](https://github.com/zarr-developers/numcodecs)
- **licensing**: MIT
- **notes**: Lossless if `astype` can represent all scaled values. Lossy otherwise. With `scale=1` this is frame-of-reference encoding (subtract offset only).
