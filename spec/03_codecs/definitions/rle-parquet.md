# rle-parquet

- **canonical-id**: `rle-parquet`
- **aliases**: `{parquet, "RLE"}`, `{parquet, "RLE_DICTIONARY"}` (index stream only)
- **awareness**: stride-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "rle-parquet",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "bit_width": {
        "type": "uint"
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
      "bit_width": {
        "type": "uint"
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
- **spec**: https://parquet.apache.org/docs/file-format/data-pages/encodings/#run-length-encoding--bit-packing-hybrid-rle--3
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
- **notes**: Hybrid RLE and bit-packing. Self-selects per block between RLE runs and bit-packed runs via header byte. Not decomposable into a pipeline of `rle → bitpack` — the per-block adaptive selection is integral to the encoding.
