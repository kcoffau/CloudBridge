# Contributing to CloudBridge

Thank you for contributing to CloudBridge! This document outlines the standards and processes for contributing to this project.

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [Branching Strategy](#branching-strategy)
3. [Commit Conventions](#commit-conventions)
4. [Pull Request Process](#pull-request-process)
5. [Review Standards](#review-standards)
6. [Architecture Decision Records (ADRs)](#architecture-decision-records-adrs)
7. [Testing Requirements](#testing-requirements)
8. [Security Considerations](#security-considerations)
9. [Release Process](#release-process)

---

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/). By participating, you are expected to uphold this code.

## Branching Strategy

### Branch Types

| Prefix | Purpose | Example |
|--------|---------|---------|
| `main` | Protected, release-ready code only | `main` |
| `phase<N>/<topic>` | Phase-based feature work | `phase0/repo-layout`, `phase1/sync-engine` |
| `fix/<issue-id>` | Bug fixes | `fix/MUL-12` |
| `docs/<topic>` | Documentation only | `docs/adr-process` |
| `chore/<topic>` | Maintenance, tooling, deps | `chore/update-deps` |

### Rules

1. **Never push directly to `main`** — all changes go through pull requests
2. **Branch from `main`** — keep branches short-lived and focused
3. **Delete merged branches** — clean up after PR merge
4. **No long-running feature branches** — rebase frequently onto `main`

### Branch Protection (main)

- Require PR review from **at least 1 different agent**
- Require **Test Manager** approval for testing-related changes
- Require **Security Manager** approval for security-related changes
- Require status checks to pass (CI, tests, lint)
- No force pushes
- No deletion

---

## Commit Conventions

We follow [Conventional Commits](https://www.conventionalcommits.org/) with project-specific scopes.

### Format

```
<type>(<scope>): <subject>

[body]

[footer]
```

### Types

| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no code change |
| `refactor` | Code change, no behavior change |
| `perf` | Performance improvement |
| `test` | Adding/modifying tests |
| `chore` | Maintenance, tooling, deps |
| `ad<r>` | Architecture Decision Record |

### Scopes

| Scope | Area |
|-------|------|
| `sync` | Sync engine (rclone, icloudpd) |
| `db` | Database layer (SQLite) |
| `api` | HTTP API / web server |
| `ui` | Web UI (React) |
| `auth` | Authentication, secrets, credentials |
| `config` | Configuration management |
| `packaging` | Proxmox CT, install scripts |
| `ci` | CI/CD pipelines |
| `docs` | Documentation |
| `adr` | Architecture Decision Records |

### Examples

```bash
feat(sync): add rclone-based iCloud Drive sync
fix(db): handle SQLite WAL mode corruption on unclean shutdown
ad<r>(001): select Go as implementation language
docs(adr): add ADR-002 sync engine decision
chore(ci): add golangci-lint to pipeline
test(api): add integration tests for /api/jobs endpoint
```

### Commit Message Guidelines

- **Subject**: Imperative mood, lowercase, no period, ≤ 50 chars
- **Body**: Explain *what* and *why*, not *how* — wrap at 72 chars
- **Footer**: Reference issues (`Fixes #123`), breaking changes (`BREAKING CHANGE:`)

---

## Pull Request Process

### PR Title Format

Same as commit subject: `<type>(<scope>): <subject>`

### PR Description Template

```markdown
## Summary
Brief description of what this PR does and why.

## Changes
- Bullet list of user-visible changes
- Include migration notes if applicable

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed (describe)

## Security
- [ ] No secrets in code/logs
- [ ] No new outbound connections
- [ ] Credentials handled per ADR-003

## Review Checklist
- [ ] Self-reviewed
- [ ] Conventional commits
- [ ] ADR updated if architecture changed
- [ ] Docs updated if user-facing change

## Related
Fixes #<issue-id>
```

### PR Size Guidance

- **Small** (< 200 lines): Preferred, faster review
- **Medium** (200–500 lines): Acceptable with clear description
- **Large** (> 500 lines): Split into multiple PRs

---

## Review Standards

### Governance Rule: No Agent Approves Own Work

> **Every pull request must be reviewed and approved by a different agent.** The author cannot approve or merge their own PR.

### Required Reviewers

| Area | Required Reviewer |
|------|-------------------|
| All PRs | At least 1 different agent |
| Testing changes | Test Manager agent |
| Security changes | Security Manager agent |
| Architecture changes | Engineering Manager agent |

### Review Criteria

1. **Correctness**: Does the code do what it claims?
2. **Security**: No secrets, proper encryption, network constraints respected
3. **Testing**: Adequate coverage, tests pass
4. **Style**: Matches Go/TypeScript conventions, passes linters
5. **Documentation**: ADRs updated, README/CONTRIBUTING current
6. **Performance**: No obvious regressions

### Review Workflow

1. Author opens PR → assigns reviewers
2. Reviewers comment, request changes, or approve
3. Author addresses feedback → pushes updates
4. **All required reviewers approve** → PR can be merged
5. Author merges (squash and merge preferred) → deletes branch

### Veto Power

- **Test Manager**: Can veto any PR affecting test strategy or coverage
- **Security Manager**: Can veto any PR affecting secrets, auth, or network boundaries
- Veto must include concrete reasoning and proposed resolution

---

## Architecture Decision Records (ADRs)

### When to Write an ADR

Any decision that:
- Affects multiple components
- Is difficult to reverse
- Has significant trade-offs
- Sets a precedent for future work

### ADR Format

Location: `docs/adr/<NNN>-<short-title>.md`

```markdown
# ADR-<NNN>: <Title>

**Status**: Proposed | Accepted | Superseded | Deprecated
**Date**: YYYY-MM-DD
**Deciders**: <agent names>
**Consulted**: <agent names>

## Context

What is the problem? What forces are at play?

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| 1. <Option A> | ... | ... |
| 2. <Option B> | ... | ... |

## Decision

What we decided and why.

## Consequences

### Positive
- ...

### Negative
- ...

### Neutral
- ...

## Follow-up

- [ ] Implementation task created
- [ ] Related ADRs: ADR-XXX, ADR-YYY
```

### ADR Lifecycle

1. **Proposed** — Draft written, circulated for comment
2. **Accepted** — Consensus reached, implementation proceeds
3. **Superseded** — Replaced by newer ADR (link to replacement)
4. **Deprecated** — No longer relevant (link to successor if any)

### Numbering

Sequential from 001. Never reuse numbers.

---

## Testing Requirements

### Test Pyramid

| Level | Target | Tools |
|-------|--------|-------|
| Unit | ≥ 80% coverage | `go test`, `vitest` |
| Integration | Key paths covered | `go test -tags=integration` |
| E2E | Critical user flows | Playwright / custom |

### Running Tests

```bash
# Unit tests (fast)
go test ./...

# Integration tests (requires deps)
go test -tags=integration ./...

# All tests with coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### Test Conventions

- Test files: `<name>_test.go` (Go), `<name>.test.ts` (TypeScript)
- Table-driven tests for multiple cases
- Mock external dependencies (rclone, SMB, iCloud)
- No real Apple credentials in tests

---

## Security Considerations

### Non-Negotiable Rules

1. **No secrets in code** — ever (not even test fixtures)
2. **No secrets in logs** — redact before output
3. **No secrets in API responses** — never return credentials
4. **Encrypt at rest** — per ADR-003 (SQLite + encryption)
5. **Network boundaries** — only Apple iCloud and configured SMB host
6. **Fine-grained tokens** — repo-scoped, least privilege

### Credentials Handling

- Stored in SQLite vault (encrypted)
- Injected at runtime via environment / config file (0600 perms)
- Never hardcoded, never committed

### Dependency Security

- `go mod verify` on every build
- `govulncheck` in CI
- Pin dependencies in `go.sum`
- Audit before adding new deps

---

## Release Process

### Versioning

[Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`

- **MAJOR**: Breaking API changes, architecture shifts
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, security patches

### Release Checklist

- [ ] All Phase gates passed
- [ ] Test Manager sign-off
- [ ] Security Manager sign-off
- [ ] CHANGELOG.md updated
- [ ] Version tag created (`vX.Y.Z`)
- [ ] Release notes published
- [ ] Proxmox install script tested

### Release Vetoes

- **Test Manager**: Insufficient test coverage or failing tests
- **Security Manager**: Unresolved security findings

---

## Getting Help

- Check existing [issues](https://github.com/kcoffau/CloudBridge/issues)
- Ask in PR comments
- Contact the Engineering Manager agent

---

*This document is a living document. Propose changes via PR.*