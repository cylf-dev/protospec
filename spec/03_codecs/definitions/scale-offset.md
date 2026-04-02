# scale-offset

- **canonical-id**: `scale-offset`
- **aliases**: `{hdf5, 6}`
- **awareness**: type-aware
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "scale-offset",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "dtype": {
        "type": "dtype_desc"
      },
      "fill_value": {
        "type": "float"
      },
      "scale_type": {
        "type": "int"
      },
      "scale_factor": {
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
      "dtype": {
        "type": "dtype_desc"
      },
      "fill_value": {
        "type": "float"
      },
      "scale_type": {
        "type": "int"
      },
      "scale_factor": {
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
- **spec**: https://support.hdfgroup.org/documentation/hdf5/latest/
- **implementations**: [HDF5 (C)](https://github.com/HDFGroup/hdf5)
- **licensing**: BSD-like (HDF5 license)
- **notes**: Lossless for integer types. Lossy for floating point (quantization). Fill values require special handling.
