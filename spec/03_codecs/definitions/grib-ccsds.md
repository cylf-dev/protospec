# grib-ccsds ⚠️

- **canonical-id**: `grib-ccsds`
- **aliases**: `{grib-edition-2, "ccsds"}`, `{grib2, "template-42"}`
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "grib-ccsds",
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
- **spec**: CCSDS 121.0-B; WMO GRIB2 template 42 ⚠️
- **implementations**: [eccodes](https://github.com/ecmwf/eccodes), [libaec](https://gitlab.dkrz.de/k202009/libaec)
- **licensing**: Apache 2.0 (eccodes); BSD (libaec)
- **notes**: ⚠️ Low confidence. Adaptive entropy coding from space mission data.
