# ccitt-group4

- **canonical-id**: `ccitt-group4`
- **aliases**: `{tiff, 4}`
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "ccitt-group4",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "image_width": {
        "type": "uint"
      },
      "t6_options": {
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
      "t6_options": {
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
- **spec**: ITU-T T.6
- **implementations**: [libtiff](https://libtiff.gitlab.io/libtiff/), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: Unencumbered
- **notes**: Bilevel (1-bit) image data only. Pure 2D encoding — each row relative to previous.
