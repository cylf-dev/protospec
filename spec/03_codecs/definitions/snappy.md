# snappy

- **canonical-id**: `snappy`
- **aliases**: `{parquet, "SNAPPY"}`, `{orc, "SNAPPY"}`, `{arrow-ipc, "SNAPPY"}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "snappy",
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
- **spec**: https://github.com/google/snappy/blob/main/format_description.txt
- **implementations**: [snappy (C++ reference)](https://github.com/google/snappy), [snap (Rust)](https://github.com/BurntSushi/rust-snappy)
- **licensing**: BSD 3-Clause
- **notes**: No HDF5 registered filter. Prioritizes speed over compression ratio. No configuration parameters.
