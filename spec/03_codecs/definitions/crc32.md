# crc32

- **canonical-id**: `crc32`
- **aliases**: `{parquet, "page-crc32"}` ⚠️ verify identifier
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "crc32",
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
- **spec**: https://www.rfc-editor.org/rfc/rfc3720#appendix-B (CRC-32 / ISO 3309)
- **implementations**: Widely available in standard libraries (zlib crc32, Python zlib.crc32)
- **licensing**: Unencumbered
- **notes**: ⚠️ Standard CRC-32 (polynomial 0x04C11DB7, ISO 3309). Parquet page headers include an optional CRC-32 checksum for page data integrity. Verify whether Parquet's CRC is appended to the page bytes (making it a codec in the pipeline sense) or stored in the page header metadata (making it a format-driver concern outside the codec layer). If the latter, this entry should be revised or removed.
