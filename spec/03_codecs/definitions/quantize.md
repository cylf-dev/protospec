# quantize

- **canonical-id**: `quantize`
- **aliases**: `{numcodecs, "numcodecs.Quantize"}`
- **awareness**: type-aware
- **lossiness**: lossy
- **signature**:
```json
{
  "codec_id": "quantize",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "digits": {
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
- **spec**: https://numcodecs.readthedocs.io/en/stable/quantize.html
- **implementations**: [numcodecs](https://github.com/zarr-developers/numcodecs)
- **licensing**: MIT
