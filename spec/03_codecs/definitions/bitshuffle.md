# bitshuffle

- **canonical-id**: `bitshuffle`
- **aliases**: `{hdf5, 32008}`, `{numcodecs, "numcodecs.BitShuffle"}`, `{blosc2, "bitshuffle"}`
- **awareness**: stride-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "bitshuffle",
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
- **spec**: https://github.com/kiyo-masui/bitshuffle
- **implementations**: [bitshuffle](https://github.com/kiyo-masui/bitshuffle), [numcodecs](https://github.com/zarr-developers/numcodecs)
- **licensing**: MIT
- **notes**: The HDF5 filter variant (32008) optionally bundles LZ4 compression as an atomic operation. In a decomposed pipeline this should be modeled as `bitshuffle → lz4`.
