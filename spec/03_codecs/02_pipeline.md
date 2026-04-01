# Pipeline Model

A pipeline is a codec whose internal logic is expressed as a directed acyclic
graph (DAG) of codec steps. A pipeline declares its interface in the same
structure as a leaf codec using `encode` and `decode` blocks with `inputs` and
`outputs`, but additionally defines the steps and wiring that implement each
direction.

Pipelines are the interface between format drivers and the Cylf pipeline engine:
a format driver constructs a pipeline definition, and the engine executes it.

Pipelines are defined in JSON.

## Pipeline Object

```json
{
  "codec_id": "simple-tiff-decode",
  "constants": {
    "bytes_per_sample": {"type": "int", "value": 2},
    "width": {"type": "int", "value": 1024}
  },
  "encode": {
    "inputs": {
      "bytes": {"type": "bytes"}
    },
    "steps": {
      "predictor": {
        "codec_id": "tiff-predictor-2",
        "inputs": {
          "bytes": "inputs.bytes",
          "bytes_per_sample": "constants.bytes_per_sample",
          "width": "constants.width"
        }
      },
      "compress": {
        "codec_id": "zlib",
        "inputs": {
          "bytes": "steps.predictor.bytes"
        }
      }
    },
    "outputs": {
      "bytes": "steps.compress.bytes"
    }
  },
  "decode": {
    "inputs": {
      "bytes": {"type": "bytes"}
    },
    "steps": {
      "decompress": {
        "codec_id": "zlib",
        "inputs": {
          "bytes": "inputs.bytes"
        }
      },
      "unpredict": {
        "codec_id": "tiff-predictor-2",
        "inputs": {
          "bytes": "steps.decompress.bytes",
          "bytes_per_sample": "constants.bytes_per_sample",
          "width": "constants.width"
        }
      }
    },
    "outputs": {
      "bytes": "steps.unpredict.bytes"
    }
  },
  "sources": {
    "tiff-predictor-2": "https://example.com/cylf/codecs/tiff/predictor/2.wasm"
  }
}
```

### `codec_id` (string, required)

The canonical identifier for this pipeline. Since a pipeline is a codec, this
serves the same role as a leaf codec's `codec_id`: it names the transformation
that this pipeline represents.

### `constants` (object, optional)

Values baked into the pipeline definition. Constants are never supplied by the
caller. Each key is a constant name; each value is a descriptor:

- `type` (string, required): determines how the value is serialized into the
  port-map. Same type vocabulary as the
  [Signature Model](01_signature.md).
- `value` (required): the constant's value.

Constants are shared across both directions. They are referenced from step
inputs using the `constants.<name>` namespace.

### `sources` (object, optional)

Maps `codec_id`s to URIs for codec artifacts. Supported URI schemes: `file://`,
`https://`, `oci://`. These are advisory fetch hints: if a `codec_id` cannot be
resolved from locally available modules, the engine may download from the URI
listed here. The engine is not required to use these URIs; it may prefer
locally cached modules, user-configured overrides, or registry lookups.

### `encode` and `decode` (objects, at least one required)

Each direction block defines the pipeline's interface and implementation for
that direction. A pipeline may define both directions or only one. A
unidirectional pipeline (e.g. one that wraps a lossy codec) defines only the
direction it supports.

Each direction block has the same structure:

#### `inputs` (object, required)

The ports that callers must supply when executing in this direction. Each key is
a port name; each value is a descriptor with the same properties as a leaf
codec's input ports (`type`, `required`, `default`). See the
[Signature Model](01_signature.md) for details.

#### `steps` (object, required)

A named map of step objects defining the DAG for this direction. Keys are step
names, which become unique identifiers used in wiring references. Order in JSON
does not matter; the engine determines execution order via topological sort.

See [Steps](#steps) below.

#### `outputs` (object, required)

Maps each output port name to the step port that produces it. The key is the
name callers use to retrieve results; the value is a wiring reference of the
form `steps.<step_name>.<port>`.

Output types are not declared here, but are derived from the referenced step's
codec signature during validation.

## A Pipeline Is a Codec

A pipeline's `encode` and `decode` blocks have the same structure as a leaf
codec's signature. Strip the pipeline-specific machinery (`steps`, `constants`,
`sources`) and resolve the output wiring references to types, and you have a
leaf codec signature.

This means:

- **Interface compatibility.** The pipeline engine can validate a pipeline
  against the same rules as a leaf codec. Any system that accepts a codec
  signature can accept a pipeline's derived signature.
- **Uniform treatment.** Format drivers, registries, and tooling do not need to
  distinguish between leaf codecs and pipelines, as both present the same
  interface.

A unidirectional pipeline (one that defines only `encode` or only `decode`)
derives a unidirectional codec signature, equivalent to a leaf codec that
implements only one direction.

## Steps

Each step invokes a codec. The step's `codec_id` resolves to a codec signature,
which defines the port names and types that the step's wiring must satisfy.

```json
"steps": {
  "decompress": {
    "codec_id": "zlib",
    "inputs": {
      "bytes": "inputs.bytes"
    }
  },
  "unpredict": {
    "codec_id": "tiff-predictor-2",
    "inputs": {
      "bytes": "steps.decompress.bytes",
      "bytes_per_sample": "constants.bytes_per_sample",
      "width": "constants.width"
    }
  }
}
```

Each key in the `steps` object is the step name. Multiple steps may share a
`codec_id` but must have distinct keys.

### `codec_id` (string, required)

The logical codec identifier. The pipeline engine maps this to a codec
implementation (native or WASM) via its resolution mechanism. The resolved
codec's signature is used for validation.

### `inputs` (object, required)

Maps each codec input port name to a wiring reference. The key is the port name
as declared in the codec signature for the active direction; the value is a
wiring reference (see [Wiring References](#wiring-references)).

The engine validates step inputs against the codec's signature for the
direction the step appears in. A step in the `encode` block is validated
against the codec's `encode.inputs`; a step in the `decode` block is validated
against the codec's `decode.inputs`. Every required input must be wired or have
a default in the codec signature.

Note that outputs are intentionally absent here: they are defined by the codec
signature, and any specification here would be redundant. Where an output of a
step is wired to the input of another, validation must ensure the output type
matches the expected input type. As stated above, when an output is wired to
the pipeline output, that pipeline output takes the type of the output as
defined by the codec signature.

## Wiring References

A wiring reference is a dot-notation string identifying the source of a value:

| Form | Meaning |
|---|---|
| `inputs.<port>` | A pipeline-level input port for this direction |
| `constants.<name>` | A pipeline-level constant |
| `steps.<step_name>.<port>` | An output port of a named step |

Step output port names correspond to the codec's output ports for the active
direction. A step in the `encode` block produces outputs matching the codec's
`encode.outputs`; a step in the `decode` block produces outputs matching the
codec's `decode.outputs`.

A step output may be referenced by multiple downstream steps (fan-out). A step
input must be wired to exactly one source.

## Both Directions Explicit

When a pipeline defines both directions, each direction's DAG is defined
independently. The `encode.steps` and `decode.steps` are separate DAGs that may
use different step names, different step orderings, and even different step
counts (though in practice they will usually be mirror images using the same
codecs).

A pipeline may also define only one direction — for example, a pipeline
wrapping a lossy codec defines only `encode`. The engine will reject attempts
to execute a unidirectional pipeline in the absent direction.

This explicitness has several consequences:

- **No inversion logic.** The engine does not need to reverse DAGs. Each
  direction is directly executable as written.
- **No direction-specific flags.** Parameters like `level` (encode-only) and
  `throw_error` (decode-only) simply appear in the direction where they are
  relevant. No `encode_only` or `decode_only` annotations are needed.
- **Independent validation.** Each direction can be validated independently
  against the codec signatures for that direction.
- **Clarity for asymmetric codecs.** Codecs like `parquet-page-v2-split` that
  have different port structures in each direction are wired naturally — each
  direction's steps reference the appropriate ports.

For simple bidirectional pipelines (linear chains of symmetric codecs), the two
direction blocks can be exact mirrors of each other. Such redundancy is
intentional as it keeps both directions visible and verifiable.

## Open Design Questions

### Constants vs inline values

The `constants` block provides named, typed values that can be referenced from
step inputs. However, the same effect could be achieved by allowing step inputs
to accept inline value descriptors directly:

```json
"unpredict": {
  "codec_id": "tiff-predictor-2",
  "inputs": {
    "bytes": "steps.decompress.bytes",
    "bytes_per_sample": {"type": "int", "value": 2},
    "width": {"type": "int", "value": 1024}
  }
}
```

Doing so would eliminate need for the `constants` block.

Given this insight, we need to determine if we should support inline values,
and if so whether we should support constants at all. The trade-offs:

- **Value reuse semantics.** Constants make shared values explicit. When two
  steps both reference `constants.bytes_per_sample`, the reader knows both
  values are the same *by definition*. With inline values, identical values at
  two sites could be coincidental or intentional — the distinction is lost.
- **Uniform input syntax.** With constants, every step input value is a wiring
  reference (a string). Without them, inputs may be either a string reference or
  an inline descriptor object, requiring a type check to distinguish the two
  forms. Constants keep the input interface uniform.
- **Indirection cost.** Constants introduce a layer of indirection: the reader
  must look up the constant definition to see the actual value. This indirection
  is not opaque — the value is right there in the pipeline — but it is not
  perfectly transparent either.

We need to consider the above and reach a consensus as to whether constants
carry their weight or should be replaced by inline values, or whether both
forms should be supported.

### Conditional pipeline steps

Some codecs require conditional logic within a pipeline. ORC's varint encoding
applies a zigzag step only for signed types. Parquet pages may use different
compression codecs determined at runtime by a value in the page header. The
current pipeline model has no mechanism for conditional step execution or
branching. This is an open design question — see the
[RFC on Format Drivers and Data Orchestration](../07_proposals/01_orchestration.md) for one
proposed approach using choice nodes at the plan level. Whether conditional
branching belongs in the codec pipeline model itself or only in the higher-level
plan format is not yet decided.

## Complete Example: Verified Zstd with Dictionary Support

This pipeline compresses data with zstd (optionally using a dictionary) and
appends a crc32c checksum. It demonstrates direction-specific parameters
(`level` in encode, `throw_error` in decode), shared parameters (`dictionary`
in both directions), and constants.

Codec signatures referenced:

```json
{
  "codec_id": "zstd",
  "encode": {
    "inputs": {
      "bytes": {"type": "bytes"},
      "level": {"type": "int", "default": 3},
      "dictionary": {"type": "bytes", "required": false}
    },
    "outputs": {
      "bytes": {"type": "bytes"}
    }
  },
  "decode": {
    "inputs": {
      "bytes": {"type": "bytes"},
      "dictionary": {"type": "bytes", "required": false}
    },
    "outputs": {
      "bytes": {"type": "bytes"}
    }
  }
}
```

```json
{
  "codec_id": "crc32c",
  "encode": {
    "inputs": {
      "bytes": {"type": "bytes"}
    },
    "outputs": {
      "bytes": {"type": "bytes"}
    }
  },
  "decode": {
    "inputs": {
      "bytes": {"type": "bytes"},
      "throw_error": {"type": "bool", "default": true}
    },
    "outputs": {
      "bytes": {"type": "bytes"}
    }
  }
}
```

Pipeline definition:

```json
{
  "codec_id": "verified-zstd",
  "constants": {
    "throw_error": {"type": "bool", "value": true}
  },
  "encode": {
    "inputs": {
      "bytes": {"type": "bytes"},
      "level": {"type": "int", "default": 3},
      "dictionary": {"type": "bytes", "required": false}
    },
    "steps": {
      "compress": {
        "codec_id": "zstd",
        "inputs": {
          "bytes": "inputs.bytes",
          "level": "inputs.level",
          "dictionary": "inputs.dictionary"
        }
      },
      "checksum": {
        "codec_id": "crc32c",
        "inputs": {
          "bytes": "steps.compress.bytes"
        }
      }
    },
    "outputs": {
      "bytes": "steps.checksum.bytes"
    }
  },
  "decode": {
    "inputs": {
      "bytes": {"type": "bytes"},
      "dictionary": {"type": "bytes", "required": false}
    },
    "steps": {
      "verify": {
        "codec_id": "crc32c",
        "inputs": {
          "bytes": "inputs.bytes",
          "throw_error": "constants.throw_error"
        }
      },
      "decompress": {
        "codec_id": "zstd",
        "inputs": {
          "bytes": "steps.verify.bytes",
          "dictionary": "inputs.dictionary"
        }
      }
    },
    "outputs": {
      "bytes": "steps.decompress.bytes"
    }
  }
}
```

Key observations:

- **The pipeline's `encode` and `decode` blocks are its codec signature.**
  Strip pipeline-specific fields and resolve output types, and you have a leaf
  codec signature with `level` and `dictionary` in encode, `dictionary` in
  decode.
- **`level` appears only in `encode`.** `zstd` only needs the level on encode
  to know how much effort it should spend optimizing the compression.
  `compress.inputs.level` is wired from `inputs.level`. `level` is an
  encode-only parameter, its absence from `decode` making that directionality
  explicit.
- **`throw_error` is a constant, not a pipeline input.** The pipeline author
  has decided this is always `true`, so it's baked in. A different pipeline
  could surface it as a decode input instead.
- **`dictionary` appears in both directions.** It is wired to the zstd step in
  both `encode.steps` and `decode.steps`. The pipeline's derived signature
  correctly shows it as an input in both directions.
- **Step names differ between directions** (`compress`/`checksum` vs
  `verify`/`decompress`). The two DAGs are independent. They use the same
  codecs but the names reflect what each step does in its direction.
- **Each direction is self-contained and independently readable.** No mental
  inversion required.

