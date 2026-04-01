# parquet-page-v2-split

- **canonical-id**: `parquet-page-v2-split`
- **aliases**: `{parquet, "data-page-v2"}`
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "parquet-page-v2-split",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "rep_length": {
        "type": "uint"
      },
      "def_length": {
        "type": "uint"
      }
    },
    "outputs": {
      "rep_bytes": {
        "type": "bytes"
      },
      "def_bytes": {
        "type": "bytes"
      },
      "value_bytes": {
        "type": "bytes"
      }
    }
  },
  "decode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "rep_length": {
        "type": "uint"
      },
      "def_length": {
        "type": "uint"
      }
    },
    "outputs": {
      "rep_bytes": {
        "type": "bytes"
      },
      "def_bytes": {
        "type": "bytes"
      },
      "value_bytes": {
        "type": "bytes"
      }
    }
  }
}
```
- **pipeline** (decode, dictionary-encoded column with zstd):
```json
{
  "codec_id": "parquet-v2-dict-decode",
  "constants": {
    "def_bit_width": {
      "type": "uint",
      "value": 1
    },
    "rep_bit_width": {
      "type": "uint",
      "value": 1
    }
  },
  "decode": {
    "inputs": {
      "data_page": {
        "type": "bytes"
      },
      "dict_page": {
        "type": "bytes"
      },
      "rep_length": {
        "type": "uint"
      },
      "def_length": {
        "type": "uint"
      },
      "index_bit_width": {
        "type": "uint"
      },
      "physical_type": {
        "type": "string"
      }
    },
    "steps": {
      "decompress_page": {
        "codec_id": "zstd",
        "inputs": {
          "bytes": "inputs.data_page"
        }
      },
      "split_page": {
        "codec_id": "parquet-page-v2-split",
        "inputs": {
          "bytes": "steps.decompress_page.bytes",
          "rep_length": "inputs.rep_length",
          "def_length": "inputs.def_length"
        }
      },
      "decode_indices": {
        "codec_id": "rle-parquet",
        "inputs": {
          "bytes": "steps.split_page.value_bytes",
          "bit_width": "inputs.index_bit_width"
        }
      },
      "dict_lookup": {
        "codec_id": "plain-dictionary",
        "inputs": {
          "indices": "steps.decode_indices.bytes",
          "dictionary": "inputs.dict_page",
          "physical_type": "inputs.physical_type"
        }
      },
      "decode_def": {
        "codec_id": "rle-parquet",
        "inputs": {
          "bytes": "steps.split_page.def_bytes",
          "bit_width": "constants.def_bit_width"
        }
      },
      "decode_rep": {
        "codec_id": "rle-parquet",
        "inputs": {
          "bytes": "steps.split_page.rep_bytes",
          "bit_width": "constants.rep_bit_width"
        }
      }
    },
    "outputs": {
      "values": "steps.dict_lookup.bytes",
      "def": "steps.decode_def.bytes",
      "rep": "steps.decode_rep.bytes"
    }
  }
}
```
Note: `def_bit_width` and `rep_bit_width` are constants for non-nested schemas; for nested schemas they become pipeline inputs derived from schema depth.
- **spec**: https://parquet.apache.org/docs/file-format/data-pages/
- **implementations**: [parquet-cpp](https://github.com/apache/arrow/tree/main/cpp/src/parquet), [parquet-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
- **notes**: `rep_length` and `def_length` from page header enable clean splitting. Preferred over v1.
