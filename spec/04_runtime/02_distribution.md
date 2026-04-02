# Codec Distribution & Registry

This document describes how codec artifacts are distributed: how they get from
codec authors to the pipeline engine that executes them.

> [!NOTE]
> This document even more immature than others in this spec. Much of it needs
> more consideration and is too tightly written towards the chonkle PoC
> implementation decisions rather than the needs of a spec document.

## Wasm Codec Distribution

### Distribution Model

Cylf takes a registry-first approach to codec distribution. Codec artifacts are
published to registries and fetched on demand by the pipeline engine when a
pipeline references a codec that is not available locally.

A pipeline definition may include a `sources` mapping that provides advisory
fetch URIs for its codecs. The pipeline engine resolves codecs through a
precedence chain: locally available modules are preferred; remote fetch is a
fallback. Downloaded artifacts are cached locally to avoid redundant network
requests.

#### Supported URI schemes

| Scheme | Description |
|---|---|
| `file://` | Local filesystem path |
| `https://` | HTTPS download (e.g. GitHub Releases asset URL) |
| `oci://` | OCI artifact in a container registry (e.g. GHCR, Docker Hub) |

`http://` (unencrypted) should be rejected by conforming implementations.

### Current Approach

Codec repositories publish `.wasm` artifacts to both **GitHub Releases** (as
release assets) and **GitHub Container Registry** (as OCI artifacts). This
provides two consumption paths:

**GitHub Releases** — simple HTTPS download, no special tooling required on the
consumer side. Download URLs follow a predictable pattern. No authentication
required for public repositories. This is the lowest-friction distribution
path.

**GHCR with OCI artifacts** — OCI-based distribution, aligned with where the
Wasm ecosystem is heading. Supports rich metadata via OCI manifests
(media types, annotations, multi-artifact packages). Anonymous pulls for public
packages. Consumers need an OCI client (e.g. `oras`) or an OCI client library.

Publishing to both is handled by a single CI workflow on each tagged release.

### Future: Wasm Component Registry (warg / wa.dev)

The Bytecode Alliance's [warg](https://warg.io/) registry protocol, with
[wa.dev](https://wa.dev/) as the public instance, is purpose-built for Wasm
component distribution. It is built around the Component Model: artifacts are
expected to be component binaries with typed WIT interfaces, and the registry
indexes interface metadata for discovery. Packages are addressed as
`namespace:name@version`.

warg / wa.dev is the natural long-term home for Component Model codecs. Cylf
codecs are Component Model components (or can be), which is exactly what the
registry is designed to distribute.

Migration is deferred until the tooling matures further. The original warg
registry implementation is no longer actively developed, with work continuing
in [`wasm-pkg-tools`](https://github.com/bytecodealliance/wasm-pkg-tools)
(`wkg` CLI). Core Wasm codecs are not Component Model components and would
need separate distribution regardless of warg maturity.

### Local Caching

Downloaded `.wasm` artifacts are cached locally to avoid redundant network
fetches. The cache is organized by codec identifier. Subsequent resolutions for
the same codec skip the download.

Wasm runtimes (e.g. Wasmtime) maintain a separate compilation cache: compiled
native machine code derived from `.wasm` binaries. With a warm compilation
cache, loading a codec module is fast (~3ms measured). The download cache and
compilation cache are orthogonal — the download cache stores portable `.wasm`
binaries, the compilation cache stores platform-specific compiled output.

### Integrity and Trust

Codec integrity can be verified at multiple levels:

- **Transport-level**: HTTPS and OCI provide transport security. OCI digests
  (content hashes) ensure artifact integrity.
- **Signature verification**: Codec artifacts may carry cryptographic signatures
  that the pipeline engine verifies before loading.
- **Sandbox isolation**: Even if a Wasm codec is compromised, the runtime's
  sandbox ensures it cannot access host memory, the filesystem, or the network.
  The blast radius of an untrusted codec is tightly constrained.

The specific signature verification scheme (key management, trust roots,
signature format) is not yet specified and is an open design area.

### Embedded Codecs

Cylf's resolution model is not limited to external registries. A codec URI
could in principle reference a byte range within a data file itself — enabling
the "decoder travels with the data" pattern where codec modules are embedded
alongside the data they encode. This would provide archival self-description
guarantees similar to F3's embedded Wasm decoders.

The exact mechanism for embedded codec references (URI format, byte-range
specification, interaction with caching) is an open design area. See
[Comparison with F3](../05_design/03_f3_comparison.md) for discussion of the trade-offs
between registry-first and embedded-first distribution.

## Native Codec Distribution

A core set of widely-used native codecs (zstd, lz4, deflate, shuffle, delta,
etc.) can be bundled with the Cylf runtime. Such codecs would be always
available without any additional installation.

Beyond the bundled set, native codecs should be installable as plugins —
discoverable by the runtime at load time without being compiled into the runtime
binary. This is important for several reasons:

- **Large or specialized codecs** (e.g. JPEG 2000, ZFP, SZ3) may have
  significant dependencies that not every deployment needs. Bundling them all
  would bloat the runtime.
- **Multiple implementations of the same codec** may coexist. A codec targeting
  GPU execution, a reference CPU implementation, and a SIMD-optimized variant
  could all be installed for the same `codec_id`. The runtime's resolution
  mechanism would select among them based on configuration or capability
  detection.
- **Third-party native codecs** should be installable without modifying the
  runtime itself.

The exact mechanism for native codec plugin distribution and discovery —
whether via system packages, a plugin directory convention, a registry protocol,
or some combination — is an open design area. The resolution mechanism would
need to handle multiple installed implementations for the same `codec_id`,
including selection by preference, capability, or explicit override.

## Options Considered and Rejected

### Language-specific package registries (npm, PyPI, Maven)

GitHub Packages hosts language-specific registries (npm, Maven, NuGet) that
have no native support for generic binary artifacts. Distributing `.wasm` files
would require wrapping them in a language-specific package format (e.g. an npm
package containing a `.wasm` file). This adds unnecessary packaging overhead
and complexity, and requires consumers to install language-specific tooling.
Not applicable.

## Summary

| Channel | Applies to | Status | Consumer tooling | Auth (public) |
|---|---|---|---|---|
| Bundled in runtime | Native (core set) | Active | None | — |
| Native plugins | Native (extended) | Open design area | TBD | — |
| GitHub Releases | Wasm | Active | HTTPS (any HTTP client) | None |
| GHCR (OCI) | Wasm | Active | OCI client (e.g. `oras`) | None |
| warg / wa.dev | Wasm (Component Model) | Planned | `wkg` CLI | None |
| Embedded in data | Wasm | Open design area | — | — |
| Language-specific registries | — | Rejected | — | — |
