# Components

`go-volumes` is a set of dependency-free Go modules. Today it ships one:

<div class="fs-grid" markdown>
<a class="fs-card" href="pool.md"><img src="../assets/fs/go-volumes-pool.png"><span><code>pool</code><br><small>CoW pooled volume manager.</small></span></a>
</div>

| Module | Import path | What it does |
|--------|-------------|--------------|
| [`pool`](pool.md) | `github.com/go-volumes/pool` | Copy-on-write pooled volume manager: volumes, immutable snapshots, writable clones, sparse raw export/import. |

Planned follow-ups: multi-device pooling / RAID, online volume grow/shrink, and
free-bitmap allocation (the current allocator is a linear scan).
