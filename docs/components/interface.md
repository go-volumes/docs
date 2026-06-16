# interface — the `Device` contract

`github.com/go-volumes/interface` (Go package `volume`) is the shared
**block-device contract** for the family. Zero dependencies — `io` only,
`CGO_ENABLED=0`.

It decouples **where** the bytes live (a pooled file volume, an S3-backed chunk
store, an NBD export, a qcow2/raw image) from the **format** layered on top
(ext4, xfs, squashfs, an OCI image).

## `Device`

```go
type Device interface {
    io.ReaderAt           // ReadAt(p []byte, off int64) (int, error)
    io.WriterAt           // WriteAt(p []byte, off int64) (int, error)
    Size() (int64, error)
    Sync() error          // durably commit buffered writes
    io.Closer
}
```

`Device` is deliberately the **intersection of what a real backing already
offers**: [`pool`](pool.md)'s `*Volume` satisfies it **verbatim**.

`ReadOnly` is the read-only subset (a read-only file, a squashfs/iso9660 blob, a
read-only NBD export, an OCI image). Any `Device` also satisfies it.

## Optional capabilities

A consumer type-asserts for these and degrades gracefully when a device does not
implement one:

| Interface | Capability |
|---|---|
| `Truncater` | resize in place — `Truncate(size int64)` |
| `Named` | stable identifier — `Name()` |
| `ReadOnlyReporter` | reports whether it rejects writes — `ReadOnly()` |
| `Discarder` | TRIM/unmap a byte range so a thin/sparse backing reclaims space |

A [go-filesystems](https://github.com/go-filesystems) driver writes its on-disk
image through a `Device`, never caring which backing — [`pool`](pool.md),
[`s3`](s3.md), [`oci`](oci.md), or a raw file — is underneath.
