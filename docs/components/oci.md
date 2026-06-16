# oci — OCI-artifact backing

`github.com/go-volumes/oci` freezes a block-volume image into an immutable,
content-addressed **OCI artifact** (push to any registry) and re-opens it as a
**read-only backing** (pull from a registry). Pure Go, `CGO_ENABLED=0`,
standard library only (plus the [`interface`](interface.md) contract).

A volume is sliced into fixed-size chunks; each chunk is sha256-addressed and
pushed **once**. Identical chunks — most importantly the all-zero holes of a
sparse image — collapse to a single blob, so a mostly-empty disk freezes to a
handful of blobs regardless of its logical size. The recovered image serves
reads by pulling chunks on demand (LRU byte cache) and **rejects every write**,
so it plugs straight into [`pool`](pool.md)'s `OpenWith` as a genuinely
immutable backing.

## Packages

| Package | Purpose |
|---|---|
| `registry` | A minimal, dependency-free OCI Distribution v2 client over `net/http`: monolithic blob upload, HEAD-based dedup, manifest get/put, and anonymous / HTTP Basic / Docker–OCI bearer-token auth. sha256 digests; every pulled blob is verified. |
| `oci` (root) | `Freeze` and `OpenReadOnly`. |

Ship a golden disk image as an OCI artifact and boot many read-only volumes from
it without copying.
