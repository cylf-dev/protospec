# jpeg-xr ⚠️

- **canonical-id**: `jpeg-xr`
- **aliases**: `{tiff, 34892}` ⚠️ verify
- **awareness**: pure-bytes
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "jpeg-xr",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "quality": {
        "type": "float",
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
- **spec**: ITU-T T.832 / ISO/IEC 29199-2
- **implementations**: [jxrlib](https://github.com/4creators/jxrlib), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: BSD (jxrlib) ⚠️ verify
- **notes**: ⚠️ Low confidence. Microsoft-developed HDR-capable image codec. Internal stages not decomposed.
