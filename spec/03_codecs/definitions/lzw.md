# lzw

- **canonical-id**: `lzw`
- **aliases**: none in target formats
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "lzw",
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
- **spec**: Welch, T. (1984). "A Technique for High-Performance Data Compression". IEEE Computer.
- **implementations**: [compress (Unix)](https://en.wikipedia.org/wiki/Compress), various
- **licensing**: Unisys patent expired; now unencumbered
- **notes**: The canonical LZW algorithm. Included to document the distinction from `lzw-tiff`. TIFF uses an incompatible variant — a generic LZW implementation will not correctly decode TIFF LZW data. See `lzw-tiff`.
