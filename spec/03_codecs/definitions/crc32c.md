# crc32c

- **canonical-id**: `crc32c`
- **aliases**: `{zarr-v3, "crc32c"}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "crc32c",
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
      },
      "error_on_fail": {
        "type": "bool",
        "default": true
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
- **spec**: https://zarr-specs.readthedocs.io/en/latest/v3/codecs/crc32c/v1.0.html
- **implementations**: [zarr-python](https://github.com/zarr-developers/zarr-python), [zarrs (Rust)](https://github.com/LDeakin/zarr_tools), [tensorstore](https://github.com/google/tensorstore)
- **licensing**: Unencumbered (algorithm); implementations vary
- **notes**: Appends 4-byte CRC-32C (Castagnoli) checksum on encode; verifies and strips on decode. CRC-32C uses a different polynomial (0x1EDC6F41) than standard CRC-32 (0x04C11DB7). Recommended by the Zarr v3 sharding spec for shard index integrity. No configuration parameters beyond the runtime `error_on_fail`.
