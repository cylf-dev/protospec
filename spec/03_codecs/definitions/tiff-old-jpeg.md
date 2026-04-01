# tiff-old-jpeg

- **canonical-id**: `tiff-old-jpeg`
- **aliases**: `{tiff, 6}`
- **awareness**: structure-aware
- **lossiness**: lossy
- **signature**:
```json
{
  "codec_id": "tiff-old-jpeg",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "jpeg_tables": {
        "type": "bytes"
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
      "jpeg_tables": {
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
- **spec**: TIFF 6.0 specification (compression=6)
- **implementations**: [libtiff](https://libtiff.gitlab.io/libtiff/), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: Unencumbered
- **notes**: Old-style JPEG in TIFF. JPEG tables stored in IFD tags, supplied as second `bytes` input. Strongly prefer new-style JPEG (compression=7).
