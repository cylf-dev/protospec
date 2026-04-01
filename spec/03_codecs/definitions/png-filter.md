# png-filter

- **canonical-id**: `png-filter`
- **aliases**: `{numcodecs, "imagecodecs.pngfilter"}` ⚠️ verify
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "png-filter",
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
      },
      "filter_type": {
        "type": "string",
        "default": "adaptive"
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
- **spec**: https://www.w3.org/TR/PNG/#9Filters
- **implementations**: [libpng](http://www.libpng.org/pub/png/libpng.html), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: Unencumbered
- **notes**: Five filter types: None, Sub, Up, Average, Paeth. On decode, filter type is read from per-scanline byte; `filter_type` only affects encoding. A genuinely separable precompression step — can be paired with any entropy coder.
