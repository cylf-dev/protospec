# tiff-predictor-3

- **canonical-id**: `tiff-predictor-3`
- **aliases**: `{tiff, "predictor=3"}`
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "tiff-predictor-3",
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
      "row_width": {
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
      "dtype": {
        "type": "string"
      },
      "element_size": {
        "type": "uint"
      },
      "row_width": {
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
- **spec**: TIFF Technical Note 3 ⚠️ verify
- **implementations**: [libtiff](https://libtiff.gitlab.io/libtiff/), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: Unencumbered
- **notes**: Byte-level shuffle specific to IEEE 754 float layout, followed by horizontal differencing, row-by-row. ⚠️ Verify exact byte shuffle and delta order.
