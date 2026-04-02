# bitround

- **canonical-id**: `bitround`
- **aliases**: `{numcodecs, "numcodecs.BitRound"}`
- **awareness**: type-aware
- **lossiness**: lossy
- **signature**:
```json
{
  "codec_id": "bitround",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "keepbits": {
        "type": "int"
      },
      "dtype": {
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
- **spec**: https://numcodecs.readthedocs.io/en/stable/bitround.html
- **implementations**: [numcodecs](https://github.com/zarr-developers/numcodecs)
- **licensing**: MIT
- **notes**: Zeroes low-order mantissa bits. Often followed by zstd or lz4.
