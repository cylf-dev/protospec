# nbit

- **canonical-id**: `nbit`
- **aliases**: `{hdf5, 5}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "nbit",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "dtype": {
        "type": "dtype_desc"
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
- **notes**: Strips padding bits based on declared precision and offset in the HDF5 dtype. Requires full HDF5 dtype descriptor. Conceptually related to `bitpack` — both reduce values to fewer bits — but `nbit` derives bit width from dtype metadata rather than taking an explicit `bit_width` parameter. An implementation could potentially delegate to `bitpack` internally.
