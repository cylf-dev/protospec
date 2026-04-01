# gzip

- **canonical-id**: `gzip`
- **aliases**: `{zarr-v2, "numcodecs.GZip"}`, `{arrow-ipc, "GZIP"}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "gzip",
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
- **spec**: https://www.rfc-editor.org/rfc/rfc1952
- **implementations**: [gzip](https://www.gnu.org/software/gzip/), [zlib](https://zlib.net)
- **licensing**: GPL (gzip tool); zlib license (zlib library implementation)
- **notes**: Gzip framing = DEFLATE + gzip header/trailer with CRC-32. Distinct from zlib framing. Not the same as `deflate` despite shared underlying algorithm.
