# ADR-002: Sync Engine — rclone with icloudpd Fallback

**Status**: Proposed
**Date**: 2026-11-18
**Deciders**: Engineering Manager
**Consulted**: Test Manager, Security Manager

## Context

CloudBridge must back up two iCloud data sources to SMB shares:

1. **iCloud Drive** — Documents, Desktop, app data (~GBs to TBs)
2. **iCloud Photos** — Full photo/video library (~GBs to TBs)

Constraints:
- Run in unprivileged Debian 13 LXC (no FUSE, no CIFS mounts)
- No outbound connections except Apple iCloud and configured SMB host
- Single binary deployment (Go)
- Scheduled, incremental backups with resume capability
- Progress tracking and error reporting via web UI
- Credentials encrypted at rest, never logged

## Options Considered

| Option | iCloud Drive | iCloud Photos | SMB | Pros | Cons |
|--------|--------------|---------------|-----|------|------|
| **1. rclone (primary) + icloudpd (Photos fallback)** | ✅ Native backend | ⚠️ Limited (read-only, no live sync) | ✅ Native backend | Mature, Go library, incremental, resume, filters, encryption, SMB native, active community | icloudpd requires Python subprocess; Photos support in rclone experimental |
| **2. rclone only** | ✅ | ❌ (experimental, unreliable) | ✅ | Single tool, Go library | Photos backup unreliable; icloudpd is purpose-built for Photos |
| **3. icloudpd only** | ❌ | ✅ | ❌ (via rclone/mount) | Purpose-built for Photos | No Drive support; no SMB; Python-only |
| **4. Custom Apple API client** | ✅ | ✅ | ✅ | Full control | Enormous effort; Apple API undocumented/private; maintenance burden; auth complexity |
| **5. Restic + rclone backend** | ✅ | ✅ (via rclone) | ✅ | Deduplication, encryption, snapshots | Adds complexity; Restic repo format overkill for mirror backup; single binary harder |

## Decision

**Primary: rclone (Go library)** for iCloud Drive, iCloud Photos (best effort), and SMB destination.
**Fallback: icloudpd (Python subprocess)** for iCloud Photos when rclone fails or for full-library integrity.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CloudBridge Sync Engine                   │
├─────────────────────────────────────────────────────────────┤
│  pkg/sync/                                                   │
│  ├── engine.go          # Orchestration, scheduling, state   │
│  ├── rclone.go          # rclone library wrapper             │
│  ├── icloudpd.go        # icloudpd subprocess manager        │
│  ├── smb.go             # SMB destination config             │
│  └── types.go           # Job, Progress, Error types         │
└─────────────────────────────────────────────────────────────┘
```

### Sync Strategies

| Source | Primary Method | Fallback | Notes |
|--------|----------------|----------|-------|
| iCloud Drive | rclone `sync` (incremental) | — | `--drive-chunk-size 64M`, `--transfers 4` |
| iCloud Photos | rclone `sync` (if stable) | icloudpd `download` | rclone Photos backend uses `icloud` remote type |
| SMB Dest | rclone `sync` to `smb://` | — | Requires SMB 2+; credentials via env |

### rclone Configuration

- **iCloud Drive remote**: `type = drive`, `scope = drive`, `root_folder_id = root`
- **iCloud Photos remote**: `type = icloud`, `service = photos` (experimental)
- **SMB remote**: `type = smb`, `host`, `share`, `user`, `pass` (from vault)

### icloudpd Fallback

Triggered when:
- rclone Photos sync returns error `backend not ready` or `permission denied`
- User explicitly selects "Full Photos Download" in UI
- Scheduled weekly full integrity check

Command: `icloudpd --username $APPLE_ID --password $APP_PWD --download-path /tmp/icloud-photos --recent 0`

Output piped to rclone `rcat` for streaming to SMB, or staged locally then synced.

## Consequences

### Positive
- Leverages battle-tested rclone for 90% of sync logic
- icloudpd handles Photos edge cases (shared albums, HEIC, Live Photos)
- Single Go binary for rclone operations; Python only for fallback
- SMB native support — no CIFS mount needed in LXC
- Incremental sync with `--modify-window` for timestamp drift
- Built-in retries, bandwidth limiting, progress stats

### Negative
- Python dependency for icloudpd (must be pre-installed in LXC template)
- Two code paths for Photos (rclone + icloudpd) — more testing surface
- rclone Photos backend is experimental; may break with Apple changes
- icloudpd requires app-specific password (not 2FA-interactive)

### Neutral
- Credentials passed via environment variables (rclone) or stdin (icloudpd)
- Both tools respect `--dry-run` for testing
- Logs sanitized — no credentials in output

## Follow-up

- [ ] Add rclone as Go module dependency (`github.com/rclone/rclone/...`)
- [ ] Package icloudpd in Proxmox CT template (Python + pip install)
- [ ] Implement sync engine abstraction (`SyncSource`, `SyncDest` interfaces)
- [ ] Add health check: verify rclone remotes configure correctly
- [ ] Define retry/backoff policy for transient Apple API errors
- [ ] Related: ADR-001 (Go enables rclone lib), ADR-003 (credentials vault)

## References

- [rclone iCloud Drive backend](https://rclone.org/drive/)
- [rclone iCloud Photos backend](https://rclone.org/icloud/)
- [rclone SMB backend](https://rclone.org/smb/)
- [icloudpd GitHub](https://github.com/icloud-photos-downloader/icloud-photos-downloader)
- [Apple App-Specific Passwords](https://support.apple.com/en-us/HT204397)