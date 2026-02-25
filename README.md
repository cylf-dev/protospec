# cylf research and development

This repo is intended to contain documentation around the design and
specification of the cylf ecosystem.

## What is cylf?

Cylf is an attempt to build a new foundational ecosystem for encoding/decoding
data. It's genesis is the recognition that all data formats are roughly the
same and contain three common things: the data in chunks, some metadata
desribing that data, and specifically metadata describing the chunk
encoding(s). With this understanding we can envision a codec library that
provides the ability to encode and decode data chunks in any format simply by
reprojecting the encoding metadata into a common representation.

The codec library aims to **make data processing work like the web - portable,
sandboxed, and trustless.**

That is, chunk decoding can be unified across any chunked data format using a
codec virtual machine (VM) architecture. Just as web browsers execute untrusted
code safely via sandboxing, datasets will be able to specify their own decoders
(including novel or proprietary codecs) that consumers can execute without
installation, compatibility concerns, or trust barriers. This foundational
shift enables:

* Cross-format tooling that works universally: as Zarr has worked to unify
  array formats, we can go a step further and support formats beyond arrays
* Frictionless sharing of experimental/proprietary encodings: as data volumes
  exponentially increase, ensuring we can design and seamlessly use more
  optimal compression techniques becomes an imperative
* The prerequisite infrastructure for format-agnostic access protocols like
  CCRP

## Components of the cylf ecosystem

Cylf is currently envisioned to have four primary ecosystem components:

* Codec specification registry
  * Define codec operation and parameters
  * Codec package registry
* Actual implementations of codec specifications
  * Allow diverse implementations of same spec
  * Packages advertise codec supported specs so they can be selected for execution
* Cylf VM and DSL/compiler
  * Secure environment in which unaudited codecs can be executed
  * Extremely minimal surface area to reduce exploit potential
  * Non-standard codecs packaged with data or downloaded on-demand
  * DSL with compiler to VM bytecode to simplify codec implementation
  * Potential to run both on CPUs and in GPU
* Cylf universal codec library
  * API allows specifying a codec pipeline for any set of encodings
  * Manages installed codec packages, executes codecs in VM

At current the VM target is WebAssembly (WASM), to as to reduce scope by
leveraging that existing work. It's unclear if that will be an effective
long-term VM solution or if a purpose-built VM will be required to realize all
the project's goals (unified CPU/GPU support is an example of a potential
blocker, but this might also be a problem resolvable via multiple codec
implementations).

## Acknowledgements

Partially supported by NASA-IMPACT VEDA project
