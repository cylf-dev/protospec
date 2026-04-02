# blosc

- **canonical-id**: `blosc`
- **aliases**: `{zarr-v2, "numcodecs.Blosc"}`, `{hdf5, 32001}`
- **awareness**: pure-bytes
- **lossiness**: lossless
- **signature**:
```json
{
  "codec_id": "blosc",
  "encode": {
    "inputs": {
      "bytes": {
        "type": "bytes"
      },
      "cname": {
        "type": "string",
        "default": "lz4"
      },
      "clevel": {
        "type": "int",
        "default": 5
      },
      "shuffle": {
        "type": "int",
        "default": 1
      },
      "blocksize": {
        "type": "int",
        "default": 0
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
- **spec**: https://github.com/Blosc/c-blosc/blob/master/README.rst
- **implementations**: [c-blosc](https://github.com/Blosc/c-blosc), [python-blosc](https://github.com/Blosc/python-blosc)
- **licensing**: BSD 3-Clause
- **notes**: Self-describing: the Blosc header embedded in the chunk carries the compressor name, shuffle mode, element size, and block size. All encode parameters are encode-only because decode reads them from the header. This makes Blosc atomic — it cannot be decomposed into a pipeline without the format driver parsing the Blosc header, which violates the principle that the chunk enters the codec pipeline as-is. In Zarr v3, the equivalent operation is decomposed into explicit shuffle + compressor pipeline steps with parameters supplied externally by the format driver. Classified as pure-bytes because no data-descriptive parameters are *required* in either direction, but the optional encode parameters `shuffle` and `blocksize` allow stride-aware operation when supplied. See the [awareness classification caveats](../01_signature.md#awareness-classification-caveats).
