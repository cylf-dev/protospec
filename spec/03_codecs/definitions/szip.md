# szip ⚠️

- **canonical-id**: `szip`
- **aliases**: `{hdf5, 4}`, `{netcdf-4, "szip"}`
- **awareness**: pure-bytes
- **lossiness**: lossless ⚠️ verify for float data
- **signature**:
```json
{
  "codec_id": "szip",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "coding": {
        "type": "int"
      },
      "pixels_per_block": {
        "type": "int"
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
      "coding": {
        "type": "int"
      },
      "pixels_per_block": {
        "type": "int"
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
- **spec**: https://support.hdfgroup.org/doc_resource/SZIP/ ⚠️
- **implementations**: [libaec](https://gitlab.dkrz.de/k202009/libaec) (open-source reimplementation), [szip (original)](https://support.hdfgroup.org/doc_resource/SZIP/)
- **licensing**: ⚠️ Original SZIP has proprietary licensing restrictions. `libaec` is BSD licensed.
- **notes**: Significant licensing history. libaec is a compatible open-source reimplementation. HDF5 bundles libaec in recent versions. Verify losslessness for all dtypes.
