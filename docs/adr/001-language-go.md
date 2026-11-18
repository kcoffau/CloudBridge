# 001: Language — Go

- Status: Accepted
- Date: 2026-11-18
- Deciders: Engineering Manager
- Technical Story: MUL-4

## Context and Problem Statement

CloudBridge must run as a single binary in unprivileged Debian 13 LXC containers with no Docker, no CIFS mounts, and minimal dependencies. The binary must embed a web UI, interface with rclone/icloudpd, manage SQLite state, and handle encrypted credentials — all while maintaining a small footprint and fast startup.

## Decision Drivers

- Single binary deployment (no runtime dependencies)
- Cross-compilation for linux/amd64, linux/arm64
- Embedded static assets (web UI)
- Native SQLite support (CGO or pure Go)
- Strong standard library for HTTP, crypto, concurrency
- Mature ecosystem for rclone integration (rclone is Go)
- Small binary size (<50MB target)
- Fast cold start (<100ms)

## Considered Options

- **Go** — Single binary, embedded assets, rclone-native, excellent SQLite drivers
- **Rust** — Single binary, but rclone FFI complex, steeper learning curve, larger team ramp
- **Python** — Not single binary, slow startup, dependency hell in LXC
- **Node.js/Bun** — Single binary possible (Bun), but rclone integration weaker, larger binary
- **Zig** — Too immature for production sync engine

## Decision Outcome

Chosen option: **Go 1.23+**, because:

- rclone is written in Go; direct library integration avoids CLI parsing overhead
- `embed` package handles web UI assets natively
- `modernc.org/sqlite` provides pure-Go SQLite (no CGO, cross-compile friendly)
- Excellent crypto/stdlib for vault implementation
- Team has Go expertise; hiring pool is large
- Proven in production sync tools (rclone, syncthing, restic)

### Positive Consequences

- Direct rclone API access for efficient sync operations
- Pure-Go SQLite enables zero-CGO cross-compilation
- Single binary <30MB with UPX compression
- Rich ecosystem for CLI frameworks (cobra), config (koanf), logging (zerolog)

### Negative Consequences

- CGO required for some SQLite drivers (mitigated by pure-Go driver)
- Garbage collection latency (mitigated by GOGC tuning, arena allocation)
- Generics still maturing (Go 1.23 has improved support)

## Pros and Cons of the Options

### Go

- Pros: Single binary, embed, rclone-native, pure-Go SQLite, fast builds, large talent pool
- Cons: GC pauses, verbose error handling, no sum types

### Rust

- Pros: Zero-cost abstractions, memory safety, excellent WASM
- Cons: rclone FFI complex, steeper learning curve, longer builds, smaller talent pool

### Python

- Pros: Rapid prototyping, rich libraries
- Cons: Not single binary, slow startup, dependency management in LXC, GIL

### Node.js/Bun

- Pros: Web UI native, TypeScript shared types
- Cons: rclone integration via CLI only, larger binary, Bun less mature

## Links

- [rclone GitHub](https://github.com/rclone/rclone)
- [modernc.org/sqlite](https://github.com/modernc/sqlite)
- [Go embed package](https://pkg.go.dev/embed)
- [ADR-002: Sync Engine](002-sync-engine-rclone-icloudpd.md)
- [ADR-003: Database](003-database-sqlite.md)