# pool

`import "github.com/go-volumes/pool"`

A pure-Go, copy-on-write **pooled volume manager** — a small, ZFS-inspired
alternative to LVM thin provisioning. No cgo, no root, no device-mapper.

A pool owns a flat array of fixed-size physical blocks backed by a single file.
**Volumes** are logical block devices carved out of the pool; **snapshots** are
immutable, reference-counted captures of a volume; **clones** are instant,
space-shared writable branches. Writes are copy-on-write — see
[Concepts](../concepts.md) for the model and the anti-LVM thesis.

## Usage

```go
p, _ := pool.Create("data.pool", 64<<20) // 64 MiB pool
defer p.Close()

vol, _ := p.CreateVolume("root", 32<<20)
vol.WriteAt(data, off)                   // copy-on-write

p.Snapshot("root", "before-upgrade")     // immutable snapshot
// ... keep writing to vol; the snapshot is undisturbed ...

snap, _ := p.OpenVolume("before-upgrade")
snap.ReadAt(buf, off)                    // original bytes

// Instant, space-shared writable clone (ZFS-style).
clone, _ := p.Clone("before-upgrade", "vm1")
clone.WriteAt(data, off)                 // diverges from the origin, CoW
```

## Mounting a filesystem on a volume

A `Volume` implements the block-backend shape the
[go-filesystems](https://github.com/go-filesystems) drivers accept, so you can
format and mount a real filesystem straight onto it:

```go
vol, _ := p.CreateVolume("root", 32<<20)
fs, _ := ext4.OpenFromDevice(vol)        // ext4 driver, no idea it's on a pool
```

!!! success "Verified end-to-end"
    A real ext4 filesystem (the `go-filesystems/ext4` driver) was run live on a
    pool volume via `OpenFromDevice`: a file written, the volume snapshotted,
    the live file overwritten and a new file added. Read back through ext4, the
    **live** volume showed the new state while the **snapshot** showed the exact
    pre-snapshot filesystem — ext4 never knew it was on a CoW pool, and the
    snapshot was fully isolated.

## Raw images (e.g. Apple Virtualization.framework / vz)

Some consumers only accept **raw disk files** and do block I/O on them at the
kernel level — they can't call a Go block backend. The pool bridges to them:

```go
clone, _ := p.Clone("golden", "vm1")
clone.ExportRawFile("vm1.raw")           // sparse raw image for vz to boot
// ... vz runs and writes vm1.raw ...
p.ImportRawFile("vm1-after", "vm1.raw")  // back under pool management; snapshot it
```

`ExportRaw` writes only allocated blocks (the rest stay holes → sparse);
`ImportRaw` leaves all-zero blocks unallocated (thin). This gives raw-only
hypervisors what they lack natively — instant golden-image clones, thin
provisioning, and safe snapshots — while they still see just a raw file.

## API surface

| Pool | |
|------|---|
| `Create(path, size)` / `Open(path)` | create / open a backing pool |
| `CreateVolume(name, size)` | new all-holes volume |
| `OpenVolume(name)` | handle to an existing volume or snapshot |
| `Snapshot(src, snap)` | immutable, refcounted capture |
| `Clone(src, name)` | writable, space-shared branch |
| `ImportRaw` / `ImportRawFile` | bring a raw image under pool management (thin) |
| `Volumes()` / `Capacity()` | list volumes / free & total blocks |
| `Sync()` / `Close()` | flush / close the backing file |

| Volume | |
|--------|---|
| `ReadAt` / `WriteAt` | block-backend I/O (CoW on write) |
| `Size` / `Name` / `ReadOnly` | metadata |
| `ExportRaw` / `ExportRawFile` | sparse raw export (the vz bridge) |
| `Sync` / `Close` / `Truncate` | block-backend compatibility |

## Status / limitations

- Single backing file (multi-device pooling / RAID is a planned follow-up).
- Fixed volume size at creation; online grow/shrink not yet implemented.
- Linear free-block search (fine for moderate pools; a free bitmap is a follow-up).
