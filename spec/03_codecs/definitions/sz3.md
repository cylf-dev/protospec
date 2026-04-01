# sz3 ⚠️

- **canonical-id**: `sz3`
- **aliases**: `{hdf5, 32024}`, `{hdf5plugin, "hdf5plugin.SZ3"}`
- **awareness**: type-aware
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "sz3",
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
      "absolute": {
        "type": "float",
        "required": false
      },
      "relative": {
        "type": "float",
        "required": false
      },
      "norm2": {
        "type": "float",
        "required": false
      },
      "peak_signal_to_noise_ratio": {
        "type": "float",
        "required": false
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
- **spec**: Zhao et al. (2021). "Optimizing Error-Bounded Lossy Compression for Scientific Data by Dynamic Spline Interpolation." IEEE ICDE. https://github.com/szcompressor/SZ3
- **implementations**: [SZ3 (C++)](https://github.com/szcompressor/SZ3), [hdf5plugin (Python)](https://github.com/silx-kit/hdf5plugin)
- **licensing**: BSD
- **notes**: ⚠️ Successor to SZ2 with a modular framework and improved spline interpolation predictor. Higher compression ratios than SZ2 in most cases. Incompatible compressed format with SZ2. Same algorithmic approach (prediction → quantization → entropy coding) but with better predictors and auto-tuning. Not decomposable into catalog codecs for the same reasons as SZ2. ⚠️ Verify exact parameter interface for the HDF5 filter.
