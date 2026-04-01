# deflate

- **canonical-id**: `deflate`
- **aliases**: `{zarr-v2, "numcodecs.Zlib"}`, `{zarr-v3, "numcodecs.zlib"}`, `{hdf5, 1}`, `{tiff, 8}`, `{tiff, 32946}`, `{parquet, "GZIP"}`, `{orc, "ZLIB"}`, `{netcdf-classic, "deflate"}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "deflate",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "level": {
        "type": "int",
        "default": 1
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
- **spec**: https://www.rfc-editor.org/rfc/rfc1951 (DEFLATE), https://www.rfc-editor.org/rfc/rfc1950 (zlib framing)
- **implementations**: [zlib](https://zlib.net), [miniz](https://github.com/richgel999/miniz), [zlib-ng](https://github.com/zlib-ng/zlib-ng)
- **licensing**: zlib license
- **notes**: "deflate" and "zlib" are often conflated. DEFLATE is the algorithm; zlib adds a 2-byte header and Adler-32 checksum. TIFF compression=8 uses zlib framing. NetCDF classic uses zlib framing. Parquet labels this `GZIP` but uses zlib framing, not gzip framing. ⚠️ Verify exact framing used per format alias. NetCDF classic has no codec layer beyond deflate applied per-variable. NetCDF-4 inherits the full HDF5 filter pipeline.
