# fsst

- **canonical-id**: `fsst`
- **aliases**: `{lance, "fsst"}`, `{duckdb, "fsst"}` ⚠️ verify DuckDB identifier
- **awareness**: type-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "fsst",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "symbol_table": {
        "type": "bytes"
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
      "symbol_table": {
        "type": "bytes"
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
- **spec**: Boncz, P., Neumann, T., & Raasveldt, M. (2020). "FSST: Fast Random Access String Compression." VLDB 2020. https://www.vldb.org/pvldb/vol13/p2649-boncz.pdf
- **implementations**: [fsst (C++ reference)](https://github.com/cwida/fsst), [lance](https://github.com/lancedb/lance)
- **licensing**: MIT ⚠️ verify
- **notes**: Compresses strings by building a 256-entry symbol table from input data, then replacing matching substrings with 1-byte codes. Escape byte (0xFF) for literals. The `symbol_table` is a data-derived input: built during encoding, required for decoding, stored in codec metadata alongside the chunk. Compare with dictionary encoding (maps whole values to codes) — FSST maps substrings to codes and is effective on high-cardinality columns. ⚠️ Verify framing conventions for symbol table storage across implementations.
