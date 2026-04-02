# lzw-tiff

- **canonical-id**: `lzw-tiff`
- **aliases**: `{tiff, 5}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "lzw-tiff",
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
- **spec**: TIFF 6.0 specification section on LZW compression
- **implementations**: [libtiff](https://libtiff.gitlab.io/libtiff/), [golang/image tiff/lzw](https://github.com/golang/image/tree/master/tiff/lzw)
- **licensing**: Unencumbered
- **notes**: TIFF's LZW is incompatible with standard LZW due to an "off by one" in code length transitions. A generic LZW codec cannot decode TIFF LZW data. Additionally, libtiff includes a backwards-compatibility decode path (`LZW_COMPAT`) for older files with reversed bit order. This is a concrete example of why format libraries cannot simply be replaced by a generic codec — the TIFF format driver must invoke `lzw-tiff` specifically.
