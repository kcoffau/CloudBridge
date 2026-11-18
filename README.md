# CloudBridge

> **Status:** Phase 0 — Engineering Foundations (in progress)

## Project Goal

CloudBridge is an open-source, self-hosted, MultCloud-style backup tool installed on Proxmox VE as a community helper script. Version 1 backs up **iCloud Drive** and **iCloud Photos** to **SMB shares** on a schedule, with a web UI, running in a single LXC container with **no outbound connections except Apple iCloud and the configured SMB host**.

## Non-Negotiable Constraints

| Constraint | Detail |
|------------|--------|
| **License** | MIT (board decision) — *LICENSE file pending board confirmation* |
| **AI Models** | OpenRouter free models only (board decision) |
| **Install Target** | Native install in unprivileged Debian 13 LXC (no Docker, no CIFS mounts) |
| **Network** | No outbound connections except Apple iCloud and configured SMB host |
| **Secrets** | Encrypted at rest, never logged, never returned by API |
| **Governance** | No agent approves own work; Test Manager & Security Manager hold release vetoes |

## Architecture Overview (Proposed)

| Layer | Technology | ADR |
|-------|------------|-----|
| Language | Go (single binary, embedded UI) | [ADR-001](docs/adr/001-language.md) |
| Sync Engine | rclone (iCloud Drive/Photos/SMB); icloudpd as Photos fallback | [ADR-002](docs/adr/002-sync-engine.md) |
| Database | SQLite (jobs, schedules, credentials vault) | [ADR-003](docs/adr/003-database.md) |

## Repository Structure

```
CloudBridge/
├── cmd/                    # Main entry points
│   └── cloudbridge/        # Primary CLI/server binary
├── pkg/                    # Shared/public packages
│   ├── sync/               # Sync engine abstractions
│   ├── db/                 # Database layer
│   ├── config/             # Configuration management
│   ├── api/                # HTTP API handlers
│   ├── auth/               # Authentication & secrets
│   └── ui/                 # Embedded web UI assets
├── internal/               # Private packages (not imported externally)
│   ├── sync/
│   ├── db/
│   ├── config/
│   ├── api/
│   ├── auth/
│   └── ui/
├── web/                    # Web UI source (React/TypeScript)
│   ├── src/
│   └── public/
├── docs/                   # Documentation
│   └── adr/                # Architecture Decision Records
├── packaging/              # Proxmox CT & install scripts
│   ├── proxmox/
│   └── scripts/
├── tests/                  # Test suites
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── .github/
│   └── workflows/          # CI/CD pipelines
├── CONTRIBUTING.md         # Contribution guidelines
├── go.mod                  # Go module definition
├── go.sum                  # Go dependencies
└── README.md               # This file
```

## Development Status

| Phase | Focus | Status |
|-------|-------|--------|
| **Phase 0** | Engineering foundations (repo, ADRs, CONTRIBUTING, task breakdown) | 🔄 In Progress |
| Phase 1 | Core sync engine, database, config, basic CLI | ⏳ Planned |
| Phase 2 | Web UI, scheduling, SMB integration, packaging | ⏳ Planned |
| Phase 3 | Hardening, testing, documentation, release | ⏳ Planned |

## Getting Started

> **Note:** Phase 0 is in progress. No runnable code exists yet.

```bash
# After Phase 1 completes:
go build -o cloudbridge ./cmd/cloudbridge
./cloudbridge --help
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines, branching strategy, commit conventions, and the ADR process.

## License

MIT License (pending board confirmation — see [issue](https://github.com/kcoffau/CloudBridge/issues/1)).