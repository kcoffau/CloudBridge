# ADR-003: Database — SQLite for State, Schedules, Credentials Vault

**Status**: Proposed
**Date**: 2026-11-18
**Deciders**: Engineering Manager
**Consulted**: Test Manager, Security Manager

## Context

CloudBridge needs persistent storage for:

1. **Job definitions** — Source, destination, schedule, filters, status
2. **Run history** — Start/end time, bytes transferred, files synced, errors
3. **Schedules** — Cron expressions, next run, timezone, enabled/disabled
4. **Credentials vault** — Apple ID, app-specific password, SMB credentials (encrypted)
5. **Configuration** — Global settings, notification webhooks, UI preferences
6. **Audit log** — Security-relevant actions (credential access, config changes)

Constraints:
- Single binary deployment (no separate DB server)
- Unprivileged LXC — limited filesystem, no systemd services
- Encrypted credentials at rest, never in logs/API responses
- ACID for job state transitions (pending → running → done/failed)
- Concurrent access: web UI (reads) + scheduler (writes) + sync engine (writes)
- Backup/restore of DB must be simple (single file)
- Debian 13 default packages preferred

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **1. SQLite (embedded)** | Single file, zero-config, ACID, embedded in Go binary, WAL mode for concurrency, easy backup (copy file), Debian packaged, no network exposure | Write concurrency limited (one writer), no horizontal scaling, schema migrations manual |
| **2. PostgreSQL** | Full SQL, concurrent writers, robust | Separate process, more resources, overkill for single-container app, network config |
| **3. BoltDB / Badger / bbolt** | Pure Go, embedded, fast KV | No SQL, query flexibility limited, migrations harder, less tooling |
| **4. JSON/YAML files** | Human-readable, simple | No ACID, no concurrency safety, corruption risk, no queries |
| **5. etcd / Consul** | Distributed, watch | Overkill, separate process, network config |

## Decision

**SQLite (with WAL mode)** as the embedded database, using the pure-Go driver `modernc.org/sqlite`.

### Schema Overview

```sql
-- Enable WAL for concurrent readers + single writer
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA foreign_keys = ON;

-- Jobs: backup definitions
CREATE TABLE jobs (
    id          TEXT PRIMARY KEY,           -- UUID
    name        TEXT NOT NULL,
    source_type TEXT NOT NULL,              -- 'icloud_drive', 'icloud_photos'
    source_cfg  TEXT NOT NULL,              -- JSON config (path, filters)
    dest_type   TEXT NOT NULL,              -- 'smb'
    dest_cfg    TEXT NOT NULL,              -- JSON config (host, share, path)
    schedule_id TEXT REFERENCES schedules(id),
    filters     TEXT,                       -- JSON: include/exclude patterns
    status      TEXT NOT NULL DEFAULT 'idle', -- 'idle', 'running', 'done', 'failed'
    created_at  INTEGER NOT NULL,           -- Unix ms
    updated_at  INTEGER NOT NULL
);

-- Runs: execution history
CREATE TABLE runs (
    id          TEXT PRIMARY KEY,           -- UUID
    job_id      TEXT NOT NULL REFERENCES jobs(id),
    started_at  INTEGER NOT NULL,
    ended_at    INTEGER,
    status      TEXT NOT NULL,              -- 'running', 'success', 'failed', 'cancelled'
    bytes_total INTEGER,
    bytes_done  INTEGER,
    files_total INTEGER,
    files_done  INTEGER,
    error_msg   TEXT,
    log_path    TEXT                        -- Path to run log (optional)
);

-- Schedules: cron-based triggers
CREATE TABLE schedules (
    id          TEXT PRIMARY KEY,           -- UUID
    name        TEXT NOT NULL,
    cron_expr   TEXT NOT NULL,              -- Standard cron with seconds
    timezone    TEXT NOT NULL DEFAULT 'UTC',
    enabled     INTEGER NOT NULL DEFAULT 1, -- 0/1
    next_run    INTEGER,                    -- Unix ms, computed
    created_at  INTEGER NOT NULL,
    updated_at  INTEGER NOT NULL
);

-- Vault: encrypted credentials (one row per credential set)
CREATE TABLE vault (
    id          TEXT PRIMARY KEY,           -- UUID
    label       TEXT NOT NULL UNIQUE,       -- 'apple_primary', 'smb_backup_nas'
    type        TEXT NOT NULL,              -- 'apple_id', 'smb', 'webhook'
    data        TEXT NOT NULL,              -- JSON encrypted blob (base64)
    nonce       TEXT NOT NULL,              -- Base64 nonce for AEAD
    created_at  INTEGER NOT NULL,
    updated_at  INTEGER NOT NULL,
    accessed_at INTEGER                     -- Last decryption time (audit)
);

-- Config: global key-value settings
CREATE TABLE config (
    key         TEXT PRIMARY KEY,
    value       TEXT NOT NULL,              -- JSON value
    description TEXT,
    updated_at  INTEGER NOT NULL
);

-- Audit log: security-relevant events
CREATE TABLE audit_log (
    id          TEXT PRIMARY KEY,           -- UUID
    timestamp   INTEGER NOT NULL,
    actor       TEXT NOT NULL,              -- 'user', 'scheduler', 'api', 'system'
    action      TEXT NOT NULL,              -- 'credential_read', 'credential_write', 'job_create', etc.
    target      TEXT,                       -- Related resource ID
    details     TEXT,                       -- JSON (no secrets)
    ip_address  TEXT                        -- For API calls
);

-- Indexes
CREATE INDEX idx_runs_job_id ON runs(job_id);
CREATE INDEX idx_runs_started_at ON runs(started_at);
CREATE INDEX idx_jobs_schedule_id ON jobs(schedule_id);
CREATE INDEX idx_audit_log_timestamp ON audit_log(timestamp);
```

### Credentials Encryption

- **Algorithm**: XChaCha20-Poly1305 (via `golang.org/x/crypto/chacha20poly1305`)
- **Key derivation**: Argon2id from master passphrase + salt (stored in config, 0600 perms)
- **Master key**: Never stored; derived at startup from env var `CLOUDBRIDGE_MASTER_KEY` or interactive prompt
- **Per-credential**: Unique random nonce (24 bytes) stored with ciphertext
- **Envelope encryption**: Optional — master key encrypts DEKs, DEKs encrypt vault rows

### Access Patterns

| Component | Access | Tables |
|-----------|--------|--------|
| Web UI (API) | Read/Write | jobs, runs, schedules, config, audit_log |
| Scheduler | Read/Write | jobs, runs, schedules |
| Sync Engine | Read/Write | jobs, runs, vault (read only) |
| Auth Middleware | Read | vault (decrypt), audit_log (write) |

### Connection Management

- Single `*sql.DB` pool (max 5 connections)
- WAL mode allows concurrent readers during write
- Short-lived transactions; no long-running transactions
- `PRAGMA busy_timeout = 5000` for contention

## Consequences

### Positive
- Zero operational overhead — single file `/var/lib/cloudbridge/db.sqlite`
- Easy backup: `cp db.sqlite db.sqlite.bak` (even live with WAL)
- Pure Go driver — no cgo, cross-compiles trivially
- Strong encryption for credentials without external deps
- SQL flexibility for queries, reporting, migrations
- Debian 13 includes sqlite3 CLI for debugging

### Negative
- Write serialization — only one writer at a time (mitigated by short transactions)
- Schema migrations manual (use `goose` or custom migration runner)
- No built-in replication (not needed for single-container)
- Vault decryption requires master key at startup (unattended reboot needs env var)

### Neutral
- File permissions: `0600` for DB, `0600` for master key file if used
- WAL files (`-wal`, `-shm`) appear alongside DB — include in backup
- Vacuum periodically (`PRAGMA auto_vacuum = INCREMENTAL`)

## Follow-up

- [ ] Add `modernc.org/sqlite` and `golang.org/x/crypto` to `go.mod`
- [ ] Implement migration runner (versioned SQL files in `internal/db/migrations/`)
- [ ] Create `pkg/db` with repositories for each table
- [ ] Implement vault service with encryption/decryption
- [ ] Add master key management (env var + optional key file)
- [ ] Define backup/restore procedure in packaging docs
- [ ] Related: ADR-001 (Go + pure Go driver), ADR-002 (sync engine reads vault)

## References

- [modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite)
- [SQLite WAL mode](https://www.sqlite.org/wal.html)
- [XChaCha20-Poly1305](https://pkg.go.dev/golang.org/x/crypto/chacha20poly1305)
- [Argon2id](https://pkg.go.dev/golang.org/x/crypto/argon2)
- [goose migration tool](https://github.com/pressly/goose)