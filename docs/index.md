# go-volumes

Pure-Go **block storage** — a copy-on-write volume **pool** plus pluggable
**backings**, with no cgo and no root.

A **pool** owns a flat array of fixed-size physical blocks; **volumes** are
logical block devices carved out of it; **snapshots** are immutable,
reference-counted captures; **clones** are instant, space-shared writable
branches. Writes are copy-on-write, so overwriting a block shared with a
snapshot allocates a fresh block and leaves the snapshot untouched — a small,
ZFS-inspired alternative to LVM thin provisioning.

A tiny **`Device` contract** (`ReadAt`/`WriteAt`/`Size`/`Sync`/`Close`) is all a
filesystem driver sees, so *where* the bytes live is pluggable: a local file,
S3-compatible **object storage**, or an immutable, content-addressed **OCI
artifact**. It pairs with [go-filesystems](https://github.com/go-filesystems):
a `Volume` is exactly the block-backend shape its ext4/xfs drivers accept, so
you can format and mount a real filesystem straight onto a pool volume.

## Components

| Module | Layer | What it does |
|--------|-------|--------------|
| [`interface`](components/interface.md) | contract | The block-device `Device` contract every layer reads and writes through. |
| [`pool`](components/pool.md) | pool | Copy-on-write volume pool — volumes, snapshots, clones; fails cleanly with `ErrPoolFull`. |
| [`s3`](components/s3.md) | backing | S3-backed block store (pool `Backing`); from-scratch AWS SigV4, stdlib only. |
| [`oci`](components/oci.md) | backing | Freeze a volume into an immutable, content-addressed OCI artifact; re-open read-only. |

## Why not LVM?

LVM thin snapshots can **corrupt** when the pool fills past a threshold: ext4 (or
XFS) overwrites in place and assumes its blocks are durable, but the thin pool
needs a *new* block for the copy-on-write — and if none is free, the write fails
mid-operation and the filesystem/snapshot is left inconsistent.

`go-volumes` is built so that can't happen — see [Concepts](concepts.md).
