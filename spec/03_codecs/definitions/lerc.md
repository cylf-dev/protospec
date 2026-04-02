# lerc

- **canonical-id**: `lerc`
- **aliases**: `{tiff, 34887}`, `{numcodecs, "numcodecs.LERC"}` ⚠️
- **awareness**: type-aware
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "lerc",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "dtype": {
        "type": "string"
      },
      "shape": {
        "type": "uint[]"
      },
      "nodata_value": {
        "type": "float",
        "required": false
      },
      "max_z_error": {
        "type": "float",
        "default": 0.0
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
      "shape": {
        "type": "uint[]"
      },
      "nodata_value": {
        "type": "float",
        "required": false
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
- **spec**: https://github.com/Esri/lerc/blob/master/doc/LERC_WhitePaper.pdf
- **implementations**: [lerc (C++, Esri)](https://github.com/Esri/lerc), [imagecodecs](https://github.com/cgohlke/imagecodecs)
- **licensing**: Apache 2.0
- **notes**: Esri geospatial codec. Natively 2D-aware. Lossless when `max_z_error=0`. Handles NoData internally. GDAL's `LERC_DEFLATE` and `LERC_ZSTD` are pipelines: `lerc → deflate` or `lerc → zstd`.
