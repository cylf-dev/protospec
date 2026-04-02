# grib-jpeg2000 ⚠️

- **canonical-id**: `grib-jpeg2000`
- **aliases**: `{grib-edition-2, "jpeg2000"}`, `{grib2, "template-40"}`
- **awareness**: structure-aware
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "grib-jpeg2000",
  "encode": {
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
- **spec**: WMO GRIB2 template 40 ⚠️
- **implementations**: [eccodes](https://github.com/ecmwf/eccodes)
- **licensing**: Apache 2.0 (eccodes)
- **notes**: ⚠️ Low confidence. JPEG2000 lossless or lossy.
