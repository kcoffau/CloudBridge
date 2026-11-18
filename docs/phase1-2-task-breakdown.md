# CloudBridge — Engineering Task Breakdown: Phases 1 & 2

> **Status**: Ready for Chief of Staff to convert to tickets
> **Dependencies**: Phase 0 foundations (ADRs, repo, CONTRIBUTING) — *complete*
> **Estimated Effort**: Phase 1 ~ 3-4 weeks, Phase 2 ~ 3-4 weeks (single engineer)

---

## Phase 1: Core Engine & Infrastructure

**Goal**: Runnable binary that can configure, schedule, and execute iCloud Drive → SMB backups with encrypted credentials.

### Epic 1.1: Project Bootstrap & CI

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P1.1.1 | Go module & deps | `go mod init`, add rclone, sqlite, crypto, cobra, koanf, zerolog, chi | 0.5d | ADR-001 |
| P1.1.2 | CI pipeline | GitHub Actions: lint (golangci-lint), test, build, govulncheck, gosec | 1d | P1.1.1 |
| P1.1.3 | Makefile / taskfile | Build, test, lint, generate, release targets | 0.5d | P1.1.1 |
| P1.1.4 | Versioning & release | Goreleaser config for linux/amd64, linux/arm64, checksums, SBOM | 1d | P1.1.2 |

### Epic 1.2: Configuration & Credentials Vault

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P1.2.1 | Config schema | Koanf-based: YAML file + env overrides, validation | 1d | P1.1.1 |
| P1.2.2 | Master key management | Argon2id from `CLOUDBRIDGE_MASTER_KEY` env or prompt; key file (0600) | 1d | ADR-003 |
| P1.2.3 | Vault package | Encrypt/decrypt credentials (XChaCha20-Poly1305), per-row nonce, audit log | 2d | P1.2.2, ADR-003 |
| P1.2.4 | Vault CLI | `cloudbridge vault set/get/list/delete <label>` for Apple ID, SMB, webhook | 1d | P1.2.3 |

### Epic 1.3: Database Layer

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P1.3.1 | SQLite connection pool | WAL mode, busy_timeout, migrations runner (goose) | 1d | ADR-003 |
| P1.3.2 | Schema migrations | v1: jobs, runs, schedules, vault, config, audit_log tables + indexes | 1d | P1.3.1 |
| P1.3.3 | Repository layer | JobsRepo, RunsRepo, SchedulesRepo, VaultRepo, ConfigRepo, AuditRepo | 2d | P1.3.2 |
| P1.3.4 | Backup/restore | `sqlite3 .backup`, automated daily, retention policy | 0.5d | P1.3.1 |

### Epic 1.4: Sync Engine Core (rclone library)

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P1.4.1 | rclone integration | Vendor rclone lib, configure iCloud Drive + SMB remotes programmatically | 2d | ADR-002, P1.1.1 |
| P1.4.2 | Sync job executor | Run rclone `Sync()` with progress callbacks, retries, bandwidth limit | 2d | P1.4.1 |
| P1.4.3 | Job state machine | pending → running → success/failed/cancelled; persist to runs table | 1d | P1.3.3, P1.4.2 |
| P1.4.4 | Dry-run & verify | `--dry-run` support, post-sync hash verification option | 1d | P1.4.2 |

### Epic 1.5: Scheduler & CLI

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P1.5.1 | Cron scheduler | Parse cron (with seconds), timezone support, next_run computation, enable/disable | 1d | P1.3.3 |
| P1.5.2 | CLI commands | `job create/list/start/stop/logs`, `schedule create/list/enable/disable`, `vault`, `config` | 2d | P1.2.4, P1.3.3, P1.5.1 |
| P1.5.3 | Daemon mode | `cloudbridge serve` — runs scheduler + HTTP API on `localhost:8080` | 1d | P1.5.1, P1.5.2 |

### Epic 1.6: Testing & Hardening (Phase 1)

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P1.6.1 | Unit tests | Vault, config, scheduler, repositories (>80% coverage) | 2d | P1.2-P1.5 |
| P1.6.2 | Integration tests | SQLite migrations, rclone sync (mocked), vault round-trip | 2d | P1.6.1 |
| P1.6.3 | Security audit | gosec, govulncheck, secret scanning, penetration test vault | 1d | P1.6.1 |
| P1.6.4 | Documentation | CLI reference, config schema, vault usage, troubleshooting | 1d | P1.6.1 |

---

## Phase 2: Web UI, iCloud Photos, Packaging

**Goal**: Web UI for job management, iCloud Photos backup via icloudpd, Proxmox install script.

### Epic 2.1: Web UI (React + TypeScript, embedded via Go embed)

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P2.1.1 | UI scaffold | Vite + React + TS + Tailwind, embedded in Go via `//go:embed` | 1d | ADR-001 |
| P2.1.2 | API layer | Chi router: `/api/jobs`, `/api/runs`, `/api/schedules`, `/api/vault`, `/api/config` | 2d | P1.5.3 |
| P2.1.3 | Auth middleware | Session cookie (HttpOnly, Secure, SameSite), CSRF, rate limit | 1d | P2.1.2 |
| P2.1.4 | Dashboard | Job list, status, last/next run, manual trigger, logs viewer | 2d | P2.1.2 |
| P2.1.5 | Job wizard | Create/edit: source picker (Drive/Photos), dest picker (SMB), filters, schedule | 2d | P2.1.4 |
| P2.1.6 | Vault UI | Add/edit/delete credentials (masked), test connection buttons | 1d | P2.1.2, P1.2.3 |
| P2.1.7 | Settings | Global config, notifications, backup retention, theme | 1d | P2.1.2 |
| P2.1.8 | Real-time updates | SSE or polling for job progress, run logs streaming | 1d | P2.1.2 |

### Epic 2.2: iCloud Photos Sync (icloudpd fallback)

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P2.2.1 | icloudpd integration | Embed static binary or vendor Python; subprocess manager with stdin/stdout piping | 2d | ADR-002 |
| P2.2.2 | Photos sync executor | Stream icloudpd output → rclone `rcat` to SMB; progress tracking | 2d | P2.2.1, P1.4.2 |
| P2.2.3 | Fallback logic | Auto-detect rclone Photos failure → trigger icloudpd; manual override in UI | 1d | P2.2.2 |
| P2.2.4 | Shared albums & metadata | Preserve album structure, HEIC/Live Photos, EXIF via icloudpd flags | 1d | P2.2.2 |

### Epic 2.3: Proxmox Packaging & Install

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P2.3.1 | CT template | Debian 13 unprivileged LXC template with Python3, rclone, icloudpd pre-installed | 1d | P2.2.1 |
| P2.3.2 | Install script | `install.sh`: downloads binary, creates user, systemd service, config dir, vault init | 1d | P2.3.1 |
| P2.3.3 | Systemd unit | `cloudbridge.service` with restart, logging, env file for master key | 0.5d | P2.3.2 |
| P2.3.4 | Post-install wizard | Interactive setup: vault init, first job, schedule, web UI access | 1d | P2.3.2, P2.1.4 |
| P2.3.5 | Uninstall/upgrade | Clean removal, backup migration, version upgrade path | 1d | P2.3.2 |

### Epic 2.4: Testing, Hardening & Docs (Phase 2)

| ID | Task | Description | Est. | Dependencies |
|----|------|-------------|------|--------------|
| P2.4.1 | E2E tests | Playwright: job create → schedule → run → verify SMB; vault CRUD | 2d | P2.1.8 |
| P2.4.2 | Load/stress test | Concurrent jobs, large file sets, network interruption recovery | 1d | P2.4.1 |
| P2.4.3 | Security review | Penetration test web UI, vault key exposure, network egress audit | 1d | P2.4.1 |
| P2.4.4 | User docs | README, admin guide, troubleshooting, FAQ, architecture diagram | 1d | P2.4.1 |
| P2.4.5 | Release artifacts | Signed binaries, SBOM, Proxmox script, checksums, release notes | 1d | P2.4.4 |

---

## Dependency Graph (Critical Path)

```
Phase 1 Critical Path:
P1.1.1 → P1.1.2 → P1.1.4
           ↓
P1.2.1 → P1.2.2 → P1.2.3 → P1.2.4
           ↓              ↓
P1.3.1 → P1.3.2 → P1.3.3 → P1.3.4
           ↓              ↓
P1.4.1 → P1.4.2 → P1.4.3 → P1.4.4
           ↓              ↓
P1.5.1 → P1.5.2 → P1.5.3
           ↓
P1.6.1 → P1.6.2 → P1.6.3 → P1.6.4  (Phase 1 Gate)

Phase 2 Critical Path:
P2.1.1 → P2.1.2 → P2.1.3 → P2.1.4 → P2.1.5 → P2.1.8
           ↓                              ↓
P2.2.1 → P2.2.2 → P2.2.3 → P2.2.4        ↓
           ↓                              ↓
P2.3.1 → P2.3.2 → P2.3.3 → P2.3.4 → P2.3.5
           ↓
P2.4.1 → P2.4.2 → P2.4.3 → P2.4.4 → P2.4.5  (Phase 2 Gate / Release)
```

---

## Resource & Risk Summary

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| rclone library API instability | Medium | High | Vendor copy rclone version; pin in go.mod |
| Apple iCloud API changes break icloudpd | Medium | High | Monitor upstream; vendor tested icloudpd version; fallback to manual download |
| SMB dialect issues with target NAS | Low | Medium | Test against common NAS (TrueNAS, Synology, QNAP); configurable dialect |
| Unprivileged LXC permission edge cases | Medium | Medium | CI test in actual unprivileged LXC; document required sysctls |
| Vault master key loss = data loss | Low | Critical | Document backup procedure; support key rotation; warn on first run |
| Web UI XSS/CSRF | Low | High | CSP headers, HttpOnly cookies, CSRF tokens, input sanitization |

---

## Gate Criteria

### Phase 1 Gate (Chief of Staff sign-off)
- [ ] Binary builds and runs on Debian 13 LXC (amd64 + arm64)
- [ ] `cloudbridge job create` → `cloudbridge job start` → syncs iCloud Drive → SMB
- [ ] Credentials encrypted in vault, never in logs/API
- [ ] Scheduler triggers jobs on cron
- [ ] Unit + integration tests pass (>80% coverage)
- [ ] Security Manager approves vault design
- [ ] Test Manager approves test strategy

### Phase 2 Gate (Release Candidate)
- [ ] Web UI accessible at `http://<ct-ip>:8080`, full job lifecycle works
- [ ] iCloud Photos backup works via icloudpd fallback
- [ ] Proxmox install script creates working CT end-to-end
- [ ] E2E tests pass in clean CT
- [ ] Security Manager approves web UI + network boundaries
- [ ] Test Manager approves E2E coverage
- [ ] Documentation complete for end users

---

## Suggested Ticket Breakdown (for Chief of Staff)

Each epic above → 1 ticket per task (P1.x.y, P2.x.y). Suggested grouping:

| Ticket Batch | Tasks | Owner Hint |
|--------------|-------|------------|
| MUL-10: Bootstrap | P1.1.1–P1.1.4 | Engineer |
| MUL-11: Config & Vault | P1.2.1–P1.2.4 | Engineer + Security review |
| MUL-12: Database | P1.3.1–P1.3.4 | Engineer |
| MUL-13: Sync Engine Core | P1.4.1–P1.4.4 | Engineer + Test review |
| MUL-14: Scheduler & CLI | P1.5.1–P1.5.3 | Engineer |
| MUL-15: Phase 1 Hardening | P1.6.1–P1.6.4 | Engineer + Test + Security |
| MUL-16: Web UI Scaffold & API | P2.1.1–P2.1.3 | Engineer |
| MUL-17: Web UI Features | P2.1.4–P2.1.8 | Engineer |
| MUL-18: iCloud Photos | P2.2.1–P2.2.4 | Engineer + Test review |
| MUL-19: Proxmox Packaging | P2.3.1–P2.3.5 | Engineer |
| MUL-20: Phase 2 Hardening & Release | P2.4.1–P2.4.5 | Engineer + Test + Security |

---

*Generated by Engineering Manager — Phase 0 complete. Ready for Chief of Staff to create tickets.*