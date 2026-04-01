# arrow-ipc-buffer

- **canonical-id**: `arrow-ipc-buffer`
- **aliases**: `{arrow-ipc, "buffer-layout"}`, `{lance, "arrow-buffer-layout"}`
- **awareness**: structure-aware
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "arrow-ipc-buffer",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "arrow_type": {
        "type": "string"
      },
      "nullable": {
        "type": "bool"
      }
    },
    "outputs": {
      "validity": {
        "type": "bytes"
      },
      "offsets": {
        "type": "bytes"
      },
      "values": {
        "type": "bytes"
      }
    }
  },
  "decode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "arrow_type": {
        "type": "string"
      },
      "nullable": {
        "type": "bool"
      }
    },
    "outputs": {
      "validity": {
        "type": "bytes"
      },
      "offsets": {
        "type": "bytes"
      },
      "values": {
        "type": "bytes"
      }
    }
  }
}
```
- **spec**: https://arrow.apache.org/docs/format/Columnar.html
- **implementations**: [arrow-cpp](https://github.com/apache/arrow), [arrow-rs](https://github.com/apache/arrow-rs)
- **licensing**: Apache 2.0
- **notes**: Output streams vary by type. Non-nullable primitive: only `values`. Nullable: `validity` + `values`. Variable-width: `validity` + `offsets` + `values`. Lance inherits this layout.
