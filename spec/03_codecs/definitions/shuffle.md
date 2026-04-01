# shuffle

- **canonical-id**: `shuffle`
- **aliases**: `{hdf5, 2}`, `{numcodecs, "numcodecs.Shuffle"}`, `{blosc2, "shuffle"}`
- **awareness**: stride-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "shuffle",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "element_size": {
        "type": "uint"
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
      "element_size": {
        "type": "uint"
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
- **spec**: https://support.hdfgroup.org/documentation/hdf5/latest/_f_i_l_t_e_r.html
- **implementations**: [HDF5 (C)](https://github.com/HDFGroup/hdf5), [numcodecs](https://github.com/zarr-developers/numcodecs)
- **licensing**: Unencumbered (algorithm); HDF5 BSD-like
