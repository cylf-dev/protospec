# bitpack

- **canonical-id**: `bitpack`
- **aliases**: `{parquet, "BIT_PACKED"}` (deprecated encoding 4), `{lance, "bitpacking"}`, `{tiff, "sub-byte-packing"}` ⚠️ verify TIFF modeling
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "bitpack",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "bit_width": {
        "type": "uint"
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
  },
  "decode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "bit_width": {
        "type": "uint"
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
- **spec**: No single canonical spec. Parquet encoding spec (encoding 4); Fastlanes paper (Afroozeh & Boncz, 2023).
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs), [lance](https://github.com/lancedb/lance) (Fastlanes-based), [bitpacking crate](https://github.com/tantivy-search/bitpacking)
- **licensing**: Unencumbered (algorithm)
- **notes**: Packs integer values into `bit_width` bits per value. Decode produces native-width integers. Parquet's `BIT_PACKED` (encoding 4) is deprecated in favor of the RLE/bit-packing hybrid. `nbit` (HDF5) performs a related operation — strips padding bits based on HDF5 dtype precision and bit offset — but requires full HDF5 dtype context; see `nbit` entry. TIFF sub-byte sample packing (e.g. 12-bit samples) is effectively `bitpack(bit_width=bits_per_sample)`. ⚠️ Verify TIFF modeling.
