# ADR-001: Implementation Language — Go

**Status**: Proposed
**Date**: 2026-11-18
**Deciders**: Engineering Manager
**Consulted**: Test Manager, Security Manager

## Context

CloudBridge is a self-hosted backup tool running in an unprivileged Debian 13 LXC container on Proxmox VE. It must:

- Produce a single binary for easy distribution/installation
- Embed a web UI (React/TypeScript) for zero-config deployment
- Interface with rclone (Go) and icloudpd (Python) as external sync tools
- Manage SQLite database for job state, schedules, and encrypted credentials
- Run with minimal dependencies (no Docker, no CIFS mounts)
- Use only OpenRouter free models for any AI features
- Have no outbound connections except Apple iCloud and configured SMB host

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **1. Go** | Single binary, excellent embedding (embed.FS), native rclone library integration, strong stdlib for HTTP/SQLite/crypto, cross-compilation, fast startup, low memory, Debian 13 packaged | Learning curve if team unfamiliar; cgo needed for some SQLite extensions |
| **2. Rust** | Single binary, memory safety, performance, good embedding | Steeper learning curve, slower compile times, less mature web framework ecosystem, rclone is Go |
| **3. Python** | Rapid development, icloudpd native, rich ecosystem | No single binary (requires venv/container), slower startup, higher memory, dependency management complexity, embedding UI harder |
| **4. Node.js / TypeScript** | Native web UI, large ecosystem | No true single binary (pkg/bun have limitations), higher memory, rclone integration via CLI only, not ideal for system-level tool |

## Decision

**Go (version 1.23+)** is selected as the implementation language.

### Rationale

1. **Single binary deployment** — Critical for Proxmox community script installation
2. **rclone integration** — rclone is written in Go; using it as a library avoids CLI parsing overhead and enables progress streaming
3. **Embedded UI** — Go 1.16+ `embed` package cleanly bundles React build output
4. **SQLite** — Excellent Go drivers (modernc.org/sqlite — pure Go, no cgo; or mattn/go-sqlite3)
5. **Cryptography** — Standard library provides all needed primitives for credential encryption
6. **Cross-compilation** — Easy CI builds for amd64/arm64
7. **Debian 13 compatibility** — Go 1.23 available in backports or via official tarball
8. **Team alignment** — Board decision constrains to OpenRouter free models; Go has good LLM SDK support

## Consequences

### Positive
- Simple installation: `curl -sSL ... | bash` downloads single binary
- No runtime dependencies on target system
- Direct rclone API access for real-time progress, error handling
- Type-safe configuration and database layers
- Fast cold start for scheduled jobs

### Negative
- Web UI requires separate build step (Vite → embed)
- icloudpd (Python) must be invoked as subprocess — not library integration
- Go generics still maturing; some patterns verbose

### Neutral
- Standard Go project layout (cmd/, pkg/, internal/) adopted
- Module path: `github.com/kcoffau/CloudBridge`

## Follow-up

- [ ] Initialize `go.mod` with Go 1.23
- [ ] Add `modernc.org/sqlite` driver (pure Go, no cgo)
- [ ] Add `golang.org/x/crypto` for credential encryption
- [ ] Set up Vite + React build outputting to `web/dist` for `//go:embed`
- [ ] Configure `goreleaser` for cross-platform binary releases
- [ ] Related: ADR-002 (sync engine), ADR-003 (database)

## References

- [Go embed package](https://pkg.go.io/embed)
- [rclone Go API](https://github.com/rclone/rclone/tree/master/lib)
- [modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite)