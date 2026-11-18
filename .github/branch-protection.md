# Branch Protection Configuration for `main`

This document specifies the required branch protection settings for the `main` branch. Configure these in GitHub Settings → Branches → Branch protection rules.

## Required Settings

### Protect matching branches
- **Branch name pattern**: `main`

### Protect matching branches (rules)

#### Pull request reviews
- ✅ **Require a pull request before merging**
  - ✅ **Required approving reviews**: `1`
  - ✅ **Dismiss stale PR approvals when new commits are pushed**
  - ✅ **Require review from Code Owners** (see CODEOWNERS)
  - ✅ **Restrict who can dismiss PR reviews** → Test Manager, Security Manager, Engineering Manager

#### Status checks
- ✅ **Require status checks to pass before merging**
  - ✅ **Require branches to be up to date before merging**
  - Required checks:
    - `ci/lint` (golangci-lint, eslint)
    - `ci/test` (unit tests)
    - `ci/build` (go build)
    - `ci/security` (govulncheck, gosec)

#### Additional restrictions
- ✅ **Require conversation resolution before merging**
- ✅ **Require signed commits** (optional, recommended)
- ✅ **Require linear history** (squash merge)
- ✅ **Do not allow bypassing the above settings**
- ✅ **Restrict who can push to matching branches** → No one (only via PR merge)
- ✅ **Allow force pushes** → No
- ✅ **Allow deletions** → No

## CODEOWNERS

Create `.github/CODEOWNERS` with:

```
# Global owners (review all PRs)
* @engineering-manager

# Test Manager owns test infrastructure
/tests/ @test-manager
/.github/workflows/ @test-manager

# Security Manager owns auth, secrets, network
/pkg/auth/ @security-manager
/internal/auth/ @security-manager
/pkg/config/ @security-manager

# Sync engine requires both Test and Security review
/pkg/sync/ @test-manager @security-manager
/internal/sync/ @test-manager @security-manager
```

## Enforcement

These settings enforce:
1. **No direct pushes to main** — all changes via PR
2. **Minimum 1 review** — from a different agent (not author)
3. **Test Manager review** — for test infra, sync engine
4. **Security Manager review** — for auth, secrets, config, sync engine
5. **CI must pass** — lint, test, build, security scans
6. **Linear history** — squash merge only
7. **Conversation resolution** — all comments addressed

## Emergency Override

In case of critical production issue:
1. Engineering Manager documents justification
2. Security Manager and Test Manager notified
3. Override logged in issue tracker
4. Post-incident review within 48 hours