# bytes

- **canonical-id**: `bytes`
- **aliases**: `{zarr-v3, "bytes"}`
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "bytes",
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
      "endian": {
        "type": "string",
        "default": "little"
      },
      "order": {
        "type": "string",
        "default": "C"
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
      "endian": {
        "type": "string",
        "default": "little"
      },
      "order": {
        "type": "string",
        "default": "C"
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
- **spec**: https://zarr-specs.readthedocs.io/en/latest/v3/codecs/bytes/v1.0.html
- **implementations**: [zarr-python](https://github.com/zarr-developers/zarr-python), [zarr-rs](https://github.com/LDeakin/zarr_tools)
- **licensing**: MIT
- **notes**: Boundary codec in Zarr v3. Explicit endianness and layout order.
