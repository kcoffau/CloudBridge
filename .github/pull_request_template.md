# Pull Request Template

## Description

<!-- Provide a clear and concise description of the changes -->

## Related Issue

<!-- Link to the issue this PR addresses -->
Fixes #

## Type of Change

<!-- Check all that apply -->
- [ ] feat: New feature
- [ ] fix: Bug fix
- [ ] docs: Documentation only
- [ ] style: Formatting, no code change
- [ ] refactor: Code restructuring
- [ ] perf: Performance improvement
- [ ] test: Adding/updating tests
- [ ] chore: Maintenance

## Testing

<!-- Describe how you tested these changes -->
- [ ] Unit tests pass (`go test ./...`)
- [ ] Integration tests pass (`go test -tags=integration ./...`)
- [ ] Manual testing performed (describe below)
- [ ] Web UI lint/typecheck pass (`cd web && npm run lint && npm run typecheck`)

### Test Plan

```
# Commands run and expected results
```

## Security

<!-- Security considerations -->
- [ ] No secrets committed
- [ ] Input validation added
- [ ] Authorization checks in place
- [ ] No SQL injection vectors
- [ ] `govulncheck ./...` passes

## Checklist

- [ ] Code follows Go/TypeScript style guides
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated (README, ADRs, CHANGELOG)
- [ ] No breaking changes (or documented in footer)
- [ ] CI checks pass

## Screenshots (if UI changes)

<!-- Add screenshots here -->

## Additional Notes

<!-- Any additional context, concerns, or decisions -->
