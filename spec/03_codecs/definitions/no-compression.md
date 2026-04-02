# no-compression

- **canonical-id**: `no-compression`
- **aliases**: `{tiff, 1}`, `{parquet, "UNCOMPRESSED"}`, `{orc, "NONE"}`, `{hdf5, "no filter"}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "no-compression",
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
- **spec**: N/A
- **implementations**: N/A — identity transform
- **notes**: Included for completeness. Zarr v3's `"bytes"` codec is a separate concept (serialization with explicit endianness) — not aliased here.
