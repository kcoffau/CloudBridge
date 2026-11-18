# 002: Sync Engine — rclone for iCloud Drive/Photos/SMB; icloudpd as Photos fallback

- Status: Accepted
- Date: 2026-11-18
- Deciders: Engineering Manager
- Technical Story: MUL-4

## Context and Problem Statement

CloudBridge must sync files between iCloud Drive, iCloud Photos, SMB shares, and local storage. The sync engine must run in unprivileged LXC with no FUSE/CIFS mounts, no Docker, and no outbound connections except Apple iCloud endpoints and configured SMB hosts. It must support incremental sync, resumable transfers, conflict resolution, and bandwidth throttling.

## Decision Drivers

- No FUSE/CIFS mounts in unprivileged LXC
- Support iCloud Drive, iCloud Photos, SMB, local
- Incremental sync with resume capability
- Conflict detection and resolution
- Bandwidth limiting and scheduling
- Single binary — minimize external dependencies
- rclone already supports all required backends
- Apple iCloud Photos API is undocumented; icloudpd is the best maintained tool

## Considered Options

- **rclone (library mode)** — Native Go library, supports all backends, incremental, resumable
- **rclone (CLI mode)** — Subprocess execution, simpler integration, less control
- **Custom iCloud implementation** — Reverse-engineer Apple APIs, high maintenance burden
- **icloudpd only** — Photos only, no Drive/SMB support
- **rsync + sshfs** — Requires FUSE/sshfs, not allowed in unprivileged LXC
- **Syncthing** — Peer-to-peer, not designed for iCloud/SMB

## Decision Outcome

Chosen option: **rclone as Go library (primary) + icloudpd as Photos fallback**, because:

- rclone library mode gives direct API access: progress callbacks, granular control, no CLI parsing
- rclone supports iCloud Drive (via `--drive-impersonate` style auth), SMB, local, SFTP, WebDAV
- icloudpd handles iCloud Photos (shared albums, live photos, metadata) better than rclone's Photos backend
- Both are Go projects; icloudpd can be invoked as subprocess or library
- Single binary: rclone compiles in; icloudpd binary can be embedded or installed as dependency

### Positive Consequences

- Unified sync engine for all backends via rclone
- icloudpd fills Photos gap (live photos, shared albums, metadata preservation)
- Incremental sync via rclone's `--track-renames`, `--modify-window`, hash-based comparison
- Resume via rclone's `--partial` and server-side copy
- Bandwidth limiting via rclone's `--bwlimit` with time-based schedules
- Conflict resolution: rclone's `--backup-dir` + custom logic for Photos

### Negative Consequences

- rclone library API not stable (mitigated by vendor copy or version pinning)
- icloudpd subprocess adds external dependency (mitigated by embedding static binary)
- Apple API changes may break icloudpd (monitor upstream, vendor fallback)
- SMB via rclone requires dialect negotiation (test against target NAS)

## Pros and Cons of the Options

### rclone library mode

- Pros: Direct API, progress streaming, no CLI overhead, unified config
- Cons: API stability, larger binary, learning curve

### rclone CLI mode

- Pros: Stable interface, simple integration, any rclone version
- Cons: Parsing overhead, no streaming progress, harder error handling

### Custom iCloud

- Pros: Full control, optimized for our use case
- Cons: High maintenance, Apple legal risk, undocumented APIs change frequently

### icloudpd only

- Pros: Best Photos support, actively maintained
- Cons: No Drive/SMB, subprocess only, Python dependency

## Links

- [rclone library docs](https://rclone.org/docs/#rclone-as-a-library)
- [rclone iCloud Drive backend](https://rclone.org/icloud/)
- [rclone SMB backend](https://rclone.org/smb/)
- [icloudpd GitHub](https://github.com/icloudpd/icloudpd)
- [ADR-001: Language](001-language-go.md)
- [ADR-003: Database](003-database-sqlite.md)