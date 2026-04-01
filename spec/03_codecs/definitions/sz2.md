# sz2 ⚠️

- **canonical-id**: `sz2`
- **aliases**: `{hdf5, 32017}`, `{hdf5plugin, "hdf5plugin.SZ"}`
- **awareness**: type-aware
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "sz2",
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
      "pointwise_relative": {
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
- **spec**: Di, S. & Cappello, F. (2016). "Fast Error-bounded Lossy HPC Data Compression with SZ." IEEE IPDPS. https://github.com/szcompressor/SZ2
- **implementations**: [SZ2 (C)](https://github.com/szcompressor/SZ2), [H5Z-SZ (HDF5 filter)](https://github.com/disheng222/H5Z-SZ), [hdf5plugin (Python)](https://github.com/silx-kit/hdf5plugin)
- **licensing**: BSD
- **notes**: ⚠️ Prediction-based lossy compressor from Argonne National Laboratory. Predicts each value from its neighbors using Lorenzo predictor or linear regression, quantizes prediction errors within the error bound, and entropy-codes the residuals (Huffman + Zstd). Designed for large-scale scientific simulation output. Supports float32, float64, and integers at arbitrary dimensionality. While the backend (Huffman + Zstd) is conceptually generic compression, the prediction-quantization-entropy pipeline is co-designed with internal intermediate formats — not decomposable into catalog codecs. Error bound parameters are encode-only — the compressed stream stores the mode and tolerance. Superseded by SZ3; cannot decode SZ3 data, and SZ3 cannot decode SZ2 data. ⚠️ Verify exact parameter set for HDF5 filter interface.
