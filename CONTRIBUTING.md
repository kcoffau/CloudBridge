# Contributing to CloudBridge

Thank you for contributing to CloudBridge. This document outlines the contribution guidelines, code style, and commit conventions.

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [Getting Started](#getting-started)
3. [Code Style](#code-style)
4. [Commit Conventions](#commit-conventions)
5. [Pull Request Process](#pull-request-process)
6. [Testing](#testing)
7. [Security](#security)
8. [Architecture Decision Records](#architecture-decision-records)

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/). By participating, you are expected to uphold this code.

## Getting Started

### Prerequisites

- Go 1.23+
- Node.js 20+ (for web UI development)
- rclone 1.68+
- icloudpd (for Photos fallback testing)
- SQLite3

### Development Setup

```bash
# Clone the repository
git clone https://github.com/CloudBridge/CloudBridge.git
cd CloudBridge

# Install Go dependencies
go mod download

# Install web UI dependencies
cd web && npm install && cd ..

# Run tests
go test ./...

# Build the binary
go build -o cloudbridge ./cmd/cloudbridge
```

## Code Style

### Go

- Follow [Effective Go](https://go.dev/doc/effective_go) and [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- Run `gofmt -s -w .` before committing
- Run `go vet ./...` and `golangci-lint run` (config in `.golangci.yml`)
- Use `go:generate` directives for code generation
- Prefer standard library over third-party packages
- Maximum line length: 120 characters
- Package comments required for all public packages

#### Naming Conventions

- Packages: lowercase, single word, no underscores (`syncengine`, not `sync_engine`)
- Interfaces: noun or `-er` suffix (`Syncer`, `Reader`)
- Constructors: `New<Type>` returning `(*Type, error)`
- Private functions: lowercase (`processJob`)
- Public functions: PascalCase (`ProcessJob`)
- Constants: PascalCase (`DefaultTimeout`)
- Error variables: `Err<Name>` (`ErrNotFound`)

#### Error Handling

- Always check errors; never use `_` for error returns in production code
- Wrap errors with `fmt.Errorf("context: %w", err)` for stack traces
- Use sentinel errors (`errors.Is`, `errors.As`) for expected failures
- Custom error types implement `error` interface with `Unwrap()`

#### Testing

- Table-driven tests for multiple cases
- Test files named `<package>_test.go`
- Mock interfaces, not concrete types
- `go test -race -count=1 ./...` for race detection
- Benchmark critical paths: `go test -bench=. -benchmem`

### Web UI (TypeScript/React)

- Follow [TypeScript ESLint recommended](https://typescript-eslint.io/rules/)
- Run `npm run lint` and `npm run typecheck` before committing
- Use functional components with hooks
- Prefer composition over inheritance
- Strict mode enabled in `tsconfig.json`
- CSS Modules for component-scoped styles

### Documentation

- All public APIs documented with Go doc comments
- ADRs for all architectural decisions (see [ADR Process](#architecture-decision-records))
- Update README.md for user-facing changes
- Comment complex logic with `//` inline comments

## Commit Conventions

We follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) with the following types:

| Type       | Description                                    |
|------------|------------------------------------------------|
| `feat`     | New feature                                    |
| `fix`      | Bug fix                                        |
| `docs`     | Documentation only changes                     |
| `style`    | Formatting, missing semicolons, etc.           |
| `refactor` | Code change that neither fixes nor adds feature|
| `perf`     | Performance improvement                        |
| `test`     | Adding or updating tests                       |
| `chore`    | Maintenance, dependencies, build changes       |
| `revert`   | Reverts a previous commit                      |

### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

- **Subject**: Imperative mood, lowercase, no period, max 72 chars
- **Body**: Explain *what* and *why*, not *how*. Wrap at 72 chars.
- **Footer**: Reference issues (`Fixes #123`), breaking changes (`BREAKING CHANGE:`)

### Examples

```
feat(sync): add incremental sync for iCloud Drive

Implements delta detection using rclone's --track-renames flag.
Reduces sync time for large libraries by 60%.

Fixes #45
```

```
fix(vault): prevent credential leakage in error logs

Error messages now redact sensitive fields before logging.
Added integration test for vault error paths.

Fixes #78
```

```
refactor(db): extract migration logic to separate package

Moves migration code from internal/db to pkg/migrate for reuse.
No behavior change.

BREAKING CHANGE: pkg/migrate now requires explicit Init() call.
```

## Pull Request Process

### Branch Naming

- Feature: `feat/<short-description>` (e.g., `feat/icloud-photos-sync`)
- Fix: `fix/<short-description>` (e.g., `fix/sqlite-lock-contention`)
- Docs: `docs/<short-description>`
- Chore: `chore/<short-description>`

### PR Requirements

1. **Title**: Follows Conventional Commits format (squash-merged)
2. **Description**: Links to issue, summarizes changes, includes test plan
3. **Review**: Minimum 1 approval from Test Manager or Security Manager
4. **Checks**: All CI checks pass (lint, test, build, security scan)
5. **No self-approval**: Author cannot approve their own PR

### Review Standards

- **Correctness**: Logic matches requirements, edge cases handled
- **Security**: No secrets, proper authz, input validation, no SQL injection
- **Performance**: No N+1 queries, efficient algorithms, bounded memory
- **Maintainability**: Clear names, small functions, low coupling, documented
- **Testing**: New code has tests; bugs have regression tests

### Merge Strategy

- Squash and merge to `main`
- Delete source branch after merge
- Linear history required (rebase before merge if needed)

## Testing

### Unit Tests

- Run with `go test ./...`
- Coverage target: ≥80% for new code
- `go test -coverprofile=coverage.out ./... && go tool cover -html=coverage.out`

### Integration Tests

- Tagged with `//go:build integration`
- Run with `go test -tags=integration ./...`
- Require rclone, icloudpd, SMB test server

### End-to-End Tests

- Located in `test/e2e/`
- Run in CI with test containers
- Tagged with `//go:build e2e`

### Benchmarks

- Critical paths benchmarked in `<package>_bench_test.go`
- Compare before/after in PR description for perf changes

## Security

### Secrets Management

- **Never** commit secrets (API keys, passwords, tokens)
- Use `internal/vault` for encrypted storage at rest
- Environment variables for runtime configuration only
- Pre-commit hook scans for secrets (`gitleaks`)

### Vulnerability Scanning

- `govulncheck ./...` in CI
- `npm audit` for web dependencies
- Dependency updates via Dependabot PRs

### Threat Model

- Unprivileged LXC: no root, no CAP_SYS_ADMIN
- Network: only Apple iCloud endpoints + configured SMB host
- Data: SQLite encrypted with age/rage; keys in vault
- API: No secret exposure in responses or logs

## Architecture Decision Records

All significant architectural decisions are documented as ADRs in `docs/adr/`.

### ADR Process

1. Create ADR file: `docs/adr/NNN-short-title.md` (next sequential number)
2. Use [MADR format](https://adr.github.io/madr/) (Markdown Architectural Decision Records)
3. Status: `Proposed` → `Accepted` | `Rejected` | `Superseded`
4. Link from PR description and update README.md index
5. Review by Test Manager and Security Manager

### ADR Template

```markdown
# NNN: Title

- Status: Proposed | Accepted | Rejected | Superseded
- Date: YYYY-MM-DD
- Deciders: <names>
- Technical Story: <issue/link>

## Context and Problem Statement

Describe the forces at play.

## Decision Drivers

- Driver 1
- Driver 2

## Considered Options

- Option 1
- Option 2

## Decision Outcome

Chosen option: "Option 1", because...

### Positive Consequences

- Benefit 1

### Negative Consequences

- Drawback 1

## Pros and Cons of the Options

### Option 1

- Pros...
- Cons...

### Option 2

- Pros...
- Cons...

## Links

- Related ADRs
- External references
```

## Release Process

1. Version bump in `go.mod` and `package.json` (semver)
2. CHANGELOG.md updated with notable changes
3. Git tag: `v<major>.<minor>.<patch>`
4. GitHub Release with artifacts
5. Homebrew tap updated (if applicable)

## Questions?

Open a GitHub Discussion or contact the Engineering Manager.