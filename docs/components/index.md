# Components

`go-volumes` is a set of dependency-free Go modules (standard library only,
`CGO_ENABLED=0`) layered around one small block-device contract:

| Module | Import path | Layer | What it does |
|--------|-------------|-------|--------------|
| [`interface`](interface.md) | `github.com/go-volumes/interface` | contract | The block-device `Device` contract (`ReadAt`/`WriteAt`/`Size`/`Sync`/`Close`) every layer reads and writes through — decoupling *where* the bytes live from the *format* on top. |
| [`pool`](pool.md) | `github.com/go-volumes/pool` | pool | Copy-on-write pooled volume manager: volumes, immutable snapshots, writable clones, sparse raw export/import. Pluggable `Backing`. |
| [`s3`](s3.md) | `github.com/go-volumes/s3` | backing | S3-backed block store satisfying pool's `Backing` — run a pool on object storage. From-scratch AWS Signature V4, no SDK. |
| [`oci`](oci.md) | `github.com/go-volumes/oci` | backing | Freeze a volume into an immutable, content-addressed OCI artifact (chunk-deduped) and re-open it as a read-only backing. |

Planned follow-ups: multi-device pooling / RAID, online volume grow/shrink, and
free-bitmap allocation (the current pool allocator is a linear scan).
