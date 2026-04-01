# zstd

- **canonical-id**: `zstd`
- **aliases**: `{zarr-v2, "numcodecs.Zstd"}`, `{zarr-v3, "numcodecs.zstd"}`, `{hdf5, 32015}`, `{parquet, "ZSTD"}`, `{orc, "ZSTD"}`, `{arrow-ipc, "ZSTD"}`, `{tiff, 50000}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "zstd",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "level": {
        "type": "int",
        "default": 3
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
- **spec**: https://facebook.github.io/zstd/zstd_manual.html
- **implementations**: [zstd (C reference)](https://github.com/facebook/zstd), [python-zstd](https://github.com/sergey-dryabzhinsky/python-zstd), [zstd-rs](https://github.com/gyscos/zstd-rs)
- **licensing**: BSD 2-Clause / GPLv2 dual license
- **notes**: Compression level is encode-only; decode does not require it. TIFF tag 50000 is self-assigned by the libtiff project; Adobe's official registration process is defunct, making libtiff the de facto registry for newer TIFF compression IDs.
