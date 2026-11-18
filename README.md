# CloudBridge

CloudBridge is a single-binary file synchronization tool for unprivileged Debian 13 LXC environments. It syncs files between iCloud Drive, iCloud Photos, SMB shares, and local storage — no Docker, no CIFS mounts, no outbound connections except Apple iCloud and configured SMB hosts.

## Architecture

- **Language**: Go (single binary, embedded UI) — see [ADR-001](docs/adr/001-language-go.md)
- **Sync Engine**: rclone for iCloud Drive/Photos/SMB; icloudpd as Photos fallback — see [ADR-002](docs/adr/002-sync-engine-rclone-icloudpd.md)
- **Database**: SQLite for state (jobs, schedules, credentials vault) — see [ADR-003](docs/adr/003-database-sqlite.md)

## Repository Structure

```
CloudBridge/
├── cmd/           # Main entry points (cloudbridge, cloudbridge-web)
├── pkg/           # Public packages (sync, db, config, vault)
├── internal/      # Private implementation details
├── web/           # Embedded web UI (React/TypeScript)
├── docs/          # Documentation and ADRs
└── CONTRIBUTING.md
```

## License

MIT License — see [LICENSE](LICENSE) for details.

## Constraints

- Native install in unprivileged Debian 13 LXC (no Docker, no CIFS mounts)
- No outbound connections except Apple iCloud and configured SMB host
- Secrets encrypted at rest, never logged, never returned by API
- OpenRouter free models only for AI features
- Fine-grained, repo-scoped tokens only