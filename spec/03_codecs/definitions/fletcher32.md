# fletcher32

- **canonical-id**: `fletcher32`
- **aliases**: `{hdf5, 3}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "fletcher32",
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
      },
      "error_on_fail": {
        "type": "bool",
        "default": true
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
- **spec**: https://support.hdfgroup.org/documentation/hdf5/latest/group___d_a_p_l.html ⚠️ verify
- **implementations**: [HDF5 (C)](https://github.com/HDFGroup/hdf5)
- **licensing**: BSD-like (HDF5 license)
- **notes**: Appends 4-byte Fletcher-32 checksum on encode; verifies and strips on decode. `error_on_fail` is a runtime parameter (not stored in data). Output is 4 bytes shorter on decode.
