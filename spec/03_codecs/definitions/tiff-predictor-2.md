# tiff-predictor-2

- **canonical-id**: `tiff-predictor-2`
- **aliases**: `{tiff, "predictor=2"}`
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "tiff-predictor-2",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
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
- **spec**: TIFF 6.0 specification, Section 14
- **implementations**: [libtiff](https://libtiff.gitlab.io/libtiff/), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: Unencumbered
- **notes**: Horizontal differencing row-by-row. First sample per row stored as-is; subsequent samples store difference from previous. Requires `row_width` to know row boundaries.
