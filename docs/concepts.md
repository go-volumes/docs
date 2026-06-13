# Concepts

## The thesis — why not LVM?

LVM thin snapshots can **corrupt** when the pool fills past a threshold: a
non-CoW filesystem like ext4 or XFS overwrites blocks in place and assumes they
are durable, but the thin pool needs a *new* block for the copy-on-write. If
none is free, the write fails mid-operation and the filesystem and its snapshots
are left inconsistent.

`go-volumes` is built so that can't happen:

- A **snapshot is immutable** — its blocks are reference-counted and never
  rewritten, so it can't be corrupted by the live volume filling up.
- When the pool is full, a copy-on-write write fails **cleanly and atomically**
  with `ErrPoolFull`, *before* any on-disk state changes. A filesystem mounted
  on the volume sees a normal `ENOSPC`/`EIO` (which ext4/XFS handle) rather than
  silent corruption.

This is the ZFS behaviour — refuse the write, keep snapshots intact — brought to
non-CoW filesystems via a block layer underneath them.

## Pools, volumes, snapshots, clones

| Term | Meaning |
|------|---------|
| **Pool** | A flat array of fixed-size physical blocks in a single backing file. |
| **Volume** | A logical block device carved out of a pool. Reads of unwritten regions return zeros (sparse). |
| **Snapshot** | An immutable, reference-counted capture of a volume at a point in time. |
| **Clone** | An instant, space-shared *writable* branch (ZFS-style). Writes to the clone or its origin diverge independently via CoW. |

## Copy-on-write

A write to a block that is owned exclusively (refcount 1) goes in place. A write
to a block shared with a snapshot or clone allocates a fresh block, copies the
old contents forward (unless the whole block is being overwritten), and drops
the reference to the old block. The allocation is the only step that can fail
when the pool is full, and it fails before any mutation — so prior data and
snapshots are never left half-written.

## Filesystem-agnostic

A `Volume` implements `ReadAt`, `WriteAt`, `Sync`, `Size`, `Truncate`, `Close`
— the same block-backend shape the
[go-filesystems](https://github.com/go-filesystems) ext4/xfs drivers accept
(`OpenFromDevice`). So you can format and mount **ext4, XFS, or any block-based
driver** straight onto a pool volume, and it composes with
[go-fde](https://github.com/go-fde) for encryption (pool → fde → fs).

!!! note
    ext4/XFS are not themselves copy-on-write — they overwrite in place. The
    snapshot safety here comes from the pool layer underneath them, not from the
    filesystem.

XFS historically had no free integrated volume manager (SGI's XLV/XVM were
proprietary), so a CoW pool under XFS fills a real gap.

## Raw images — the vz bridge

Some consumers only accept **raw disk files** and do block I/O on them at the
kernel level — they can't call a Go block backend. Apple's
Virtualization.framework (vz) is the prime example. The pool bridges to them:
keep golden images + thin CoW clones + snapshots in the pool, then **export a
clone to a sparse raw file** for the consumer, and **import the modified file
back** to capture changes. Only allocated blocks are written on export and
all-zero blocks stay holes on import, so images stay thin in both directions.
