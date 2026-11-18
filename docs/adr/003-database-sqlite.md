# 003: Database — SQLite for state (jobs, schedules, credentials vault)

- Status: Accepted
- Date: 2026-11-18
- Deciders: Engineering Manager
- Technical Story: MUL-4

## Context and Problem Statement

CloudBridge needs persistent state for: sync jobs (status, progress, history), schedules (cron-like timing), and credentials vault (encrypted Apple IDs, SMB passwords, API tokens). The database must be embedded (no separate process), support encryption at rest, handle concurrent access from sync engine and web UI, and survive container restarts.

## Decision Drivers

- Embedded, zero-configuration, single file
- Encryption at rest (secrets never in plaintext)
- Concurrent readers/writers (sync engine + web API)
- ACID transactions for job state consistency
- Cross-compilation friendly (pure Go preferred)
- Small footprint (<10MB library)
- Mature, well-tested, battle-tested in production
- Backup via file copy (VACUUM into backup file)

## Considered Options

- **SQLite (modernc.org/sqlite — pure Go)** — Embedded, pure Go, encryption via SQLCipher or application-layer
- **SQLite (mattn/go-sqlite3 — CGO)** — Standard, but requires CGO, cross-compile friction
- **BoltDB / bbolt** — Pure Go, key-value, no SQL, manual indexing
- **BadgerDB** — LSM tree, pure Go, more complex, overkill for relational data
- **PostgreSQL / MySQL** — Requires separate process, not embedded
- **Redis** — Requires separate process, not durable enough for job state
- **Filesystem (JSON/YAML)** — No concurrency, no transactions, corruption risk

## Decision Outcome

Chosen option: **SQLite with `modernc.org/sqlite` (pure Go driver) + application-layer encryption via age/rage**, because:

- Pure Go driver enables zero-CGO cross-compilation for linux/amd64, linux/arm64
- SQL simplifies queries for jobs, schedules, audit logs
- Application-layer encryption: encrypt sensitive columns (credentials) before write, decrypt on read
- age/rage provides modern, audited encryption (ChaCha20-Poly1305)
- Single file: easy backup, copy, inspect with `sqlite3` CLI
- WAL mode enables concurrent readers + single writer
- Mature: used in production by rclone, syncthing, restic, GitHub CLI

### Positive Consequences

- Zero external dependencies; single binary deployment
- Cross-compile from any host (no CGO toolchain needed)
- Column-level encryption: only secrets encrypted, metadata queryable
- WAL mode: web UI reads don't block sync engine writes
- Backup: `sqlite3 cloudbridge.db ".backup backup.db"` or file copy in WAL mode
- Schema migrations via Goose or custom migrator in `pkg/migrate`

### Negative Consequences

- Application-layer encryption requires careful key management (vault package)
- SQLite write throughput limited to single writer (mitigated by batching, WAL)
- No built-in replication (not needed; single-node LXC)
- Schema changes require migrations (standard practice)

## Pros and Cons of the Options

### modernc.org/sqlite (pure Go)

- Pros: Zero CGO, cross-compile, active maintenance, passes SQLite test suite
- Cons: Slightly slower than CGO version (~10-15%), larger binary

### mattn/go-sqlite3 (CGO)

- Pros: Standard, fastest, widely used
- Cons: CGO required, cross-compile needs docker/multi-arch toolchain

### BoltDB/bbolt

- Pros: Pure Go, simple, fast for key-value
- Cons: No SQL, manual indexes, no ad-hoc queries, schema evolution harder

### BadgerDB

- Pros: Pure Go, LSM tree, good write throughput
- Cons: Overkill, complex, no SQL, larger binary

### PostgreSQL/MySQL

- Pros: Full SQL, replication, concurrent writes
- Cons: Separate process, config, memory, not embedded

## Links

- [modernc.org/sqlite](https://github.com/modernc/sqlite)
- [SQLite WAL mode](https://www.sqlite.org/wal.html)
- [age encryption](https://github.com/FiloSottile/age)
- [rage (Rust age)](https://github.com/str4d/rage)
- [Goose migrations](https://github.com/pressly/goose)
- [ADR-001: Language](001-language-go.md)
- [ADR-002: Sync Engine](002-sync-engine-rclone-icloudpd.md)