# ccitt-group3

- **canonical-id**: `ccitt-group3`
- **aliases**: `{tiff, 3}`
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "ccitt-group3",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "image_width": {
        "type": "uint"
      },
      "t4_options": {
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
      "image_width": {
        "type": "uint"
      },
      "t4_options": {
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
- **spec**: ITU-T T.4
- **implementations**: [libtiff](https://libtiff.gitlab.io/libtiff/), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: Unencumbered
- **notes**: Bilevel (1-bit) image data only. Row-oriented.
