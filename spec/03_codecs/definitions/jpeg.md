# jpeg

- **canonical-id**: `jpeg`
- **aliases**: `{tiff, 7}`, `{numcodecs, "imagecodecs.jpeg8"}` ⚠️ verify
- **awareness**: pure-bytes
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "jpeg",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "quality": {
        "type": "int",
        "default": 75
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
- **spec**: ITU-T T.81 / ISO/IEC 10918-1
- **implementations**: [libjpeg-turbo](https://github.com/libjpeg-turbo/libjpeg-turbo), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: Unencumbered (format); libjpeg-turbo BSD/zlib
- **notes**: Internally applies DCT → quantization → Huffman coding. Internal stages are not decomposed. TIFF compression=7 is new-style JPEG with self-contained strips; see `tiff-old-jpeg` (compression=6) for legacy variant.
