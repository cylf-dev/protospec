# zfp

- **canonical-id**: `zfp`
- **aliases**: `{hdf5, 32013}`, `{hdf5plugin, "hdf5plugin.Zfp"}` ⚠️ verify numcodecs identifier
- **awareness**: type-aware
- **lossiness**: conditional
- **signature**:
```json
{
  "codec_id": "zfp",
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
      "mode": {
        "type": "string"
      },
      "rate": {
        "type": "float",
        "required": false
      },
      "precision": {
        "type": "int",
        "required": false
      },
      "accuracy": {
        "type": "float",
        "required": false
      },
      "minbits": {
        "type": "int",
        "required": false
      },
      "maxbits": {
        "type": "int",
        "required": false
      },
      "maxprec": {
        "type": "int",
        "required": false
      },
      "minexp": {
        "type": "int",
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
      },
      "mode": {
        "type": "string"
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
- **spec**: Lindstrom, P. (2014). "Fixed-Rate Compressed Floating-Point Arrays." IEEE TVCG 20(12). https://zfp.readthedocs.io/
- **implementations**: [zfp (C/C++, LLNL)](https://github.com/LLNL/zfp), [H5Z-ZFP (HDF5 filter)](https://github.com/LLNL/H5Z-ZFP), [hdf5plugin (Python)](https://github.com/silx-kit/hdf5plugin)
- **licensing**: BSD 3-Clause
- **notes**: Transform-based lossy compressor for spatially correlated floating-point and integer arrays. Partitions data into 4^d blocks, applies block-floating-point conversion, a custom near-orthogonal decorrelation transform (similar to DCT), coefficient reordering, and embedded bitplane coding with truncation. Supports 1D–4D arrays of float32, float64, int32, int64; works best on 2D+ data with spatial correlation. Five compression modes selected by `mode`: `rate` (fixed bits per value), `precision` (fixed bit planes), `accuracy` (absolute error tolerance), `expert` (direct control of minbits/maxbits/maxprec/minexp), `reversible` (lossless). Only the parameters relevant to the selected mode are used. All mode-specific parameters are encode-only — the compressed stream is self-describing for decompression. Internal stages are tightly integrated and not decomposable — truncation-based coding is fundamental to the algorithm and cannot be separated from the transform.
