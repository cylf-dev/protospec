# lz4

- **canonical-id**: `lz4`
- **aliases**: `{zarr-v2, "numcodecs.LZ4"}`, `{hdf5, 32004}`, `{parquet, "LZ4_RAW"}`, `{orc, "LZ4"}`, `{blosc2, "lz4"}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "lz4",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "acceleration": {
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
- **spec**: https://github.com/lz4/lz4/blob/dev/doc/lz4_Frame_format.md
- **implementations**: [lz4 (C reference)](https://github.com/lz4/lz4), [lz4-rs](https://github.com/10xGenomics/lz4)
- **licensing**: BSD 2-Clause
- **notes**: Parquet previously used a framed LZ4 variant (`LZ4`) that was inconsistently implemented; `LZ4_RAW` is the unframed standard and preferred in newer Parquet writers. ⚠️ Verify HDF5 filter 32004 uses framed or raw variant.
