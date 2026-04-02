# grib-grid-packing ⚠️

- **canonical-id**: `grib-grid-packing`
- **aliases**: `{grib-edition-1, "simple-packing"}`
- **awareness**: structure-aware
- **lossiness**: lossy
- **signature**:
```json
{
  "codec_id": "grib-grid-packing",
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
- **spec**: https://library.wmo.int/records/item/35625-manual-on-codes-volume-i-2 ⚠️
- **implementations**: [eccodes](https://github.com/ecmwf/eccodes), [cfgrib](https://github.com/ecmwf/cfgrib)
- **licensing**: Apache 2.0 (eccodes)
- **notes**: ⚠️ Low confidence. Scale factor and reference value in GRIB headers. Verify details.
