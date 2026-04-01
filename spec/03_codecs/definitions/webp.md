# webp

- **canonical-id**: `webp`
- **aliases**: `{tiff, 50001}`
- **awareness**: pure-bytes
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "webp",
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
- **spec**: https://developers.google.com/speed/webp/docs/riff_container
- **implementations**: [libwebp](https://chromium.googlesource.com/webm/libwebp), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: BSD 3-Clause
- **notes**: Lossy mode based on VP8 intra-frame coding. Lossless mode uses spatial prediction + LZ77+Huffman. Internal stages not decomposed. TIFF tag 50001 is self-assigned by libtiff.
