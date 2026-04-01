# jpeg-xl

- **canonical-id**: `jpeg-xl`
- **aliases**: `{tiff, 50002}` ⚠️ verify tag number, `{gdal-gtiff, "JXL"}`
- **awareness**: pure-bytes
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "jpeg-xl",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "distance": {
        "type": "float",
        "default": 1.0
      },
      "effort": {
        "type": "int",
        "default": 7
      },
      "lossless": {
        "type": "bool",
        "default": false
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
- **spec**: https://jpeg.org/jpegxl/ ; ISO/IEC 18181
- **implementations**: [libjxl (C++ reference)](https://github.com/libjxl/libjxl), [jxl-rs](https://github.com/libjxl/jxl-rs), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: BSD 3-Clause (libjxl); royalty-free format
- **notes**: Supports both lossy and lossless compression. Notably supports lossless JPEG recompression. Internal stages (VarDCT, modular prediction, ANS entropy coding) are not decomposed — the intermediate representations are JXL-internal and not usefully recombineable with other codec steps. TIFF tag number needs verification.
