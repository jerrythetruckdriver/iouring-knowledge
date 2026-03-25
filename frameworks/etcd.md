# etcd and io_uring

## Status: Not Used

Zero io_uring references in the etcd codebase. Written in Go, uses bbolt (B+tree) for storage. Same story as every Go project.

## Why Not

1. **Go runtime** — goroutines + netpoller (epoll). No io_uring integration in Go's runtime.
2. **bbolt** — mmap-based B+tree. Read path is page faults, write path is fdatasync. Neither benefits enough to justify cgo complexity.
3. **Raft consensus** — latency dominated by network RTT between nodes, not local I/O syscall overhead.
4. **WAL writes** — sequential append + fsync. One syscall per batch. io_uring's linked write+fsync saves one syscall per commit — measurable but not transformative when network consensus is the bottleneck.

## Where io_uring Could Theoretically Help

- **WAL write+fsync chain** — linked ops, one submission instead of two syscalls
- **Snapshot transfer** — splice for large snapshot streaming
- **Compaction** — batch reads during bbolt compaction

But the Go runtime prevents any of this without cgo, and cgo overhead likely eats the gains.

## The Pattern

etcd joins CockroachDB, Consul, and every other Go distributed system: the runtime's I/O model is the ceiling. If you want io_uring-level storage performance in a consensus system, look at systems written in C++ (like Redpanda) or Zig (like TigerBeetle).

## See Also

- [Go + cgo Analysis](go-cgo.md) — overhead analysis for Go + io_uring
- [Database Patterns](../patterns/database-patterns.md) — WAL write+fsync chains
- [CockroachDB & Tarantool](cockroachdb-tarantool.md) — same Go limitation story
