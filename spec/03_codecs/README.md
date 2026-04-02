# Codec Specifications

Specifications for how codecs describe their interfaces and how they compose
into pipelines.

- [Signature Model](01_signature.md) — how codecs describe their interfaces:
  typed ports, bidirectional encode/decode blocks, awareness taxonomy, and
  format alias mappings
- [Pipeline Model](02_pipeline.md) — how codecs compose into executable DAGs:
  steps, wiring references, constants, explicit bidirectional definitions
- [Codec Inventory](03_inventory.md) — catalog of 60+ codecs across formats,
  with signatures, aliases, and composition properties
