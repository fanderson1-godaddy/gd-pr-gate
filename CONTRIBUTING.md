# Contributing

## Making Changes

The workflow file `.github/workflows/gate.yml` is hand-authored (not generated). Changes should:

1. **Preserve backwards compatibility** — callers use `@v1` and expect stable behavior
2. **Follow existing patterns** — match the style of other gdcorp-actions workflows
3. **Update documentation** — sync changes to `docs/` files
4. **Test thoroughly** — use throwaway PRs on a test repo

## Testing Changes

Before merging:

1. Create a feature branch in this repo
2. Point a test caller at your branch:
   ```yaml
   uses: gdcorp-actions/gd-pr-gate/.github/workflows/gate.yml@your-branch-name
   ```
3. Open throwaway PRs with:
   - Clean change (expect ✅)
   - Vulnerable dep (expect ❌)
   - Semgrep trigger (expect ❌)
4. Verify comment updates, check status, and step summary

## Branching

- `main` — stable, production-ready
- `feat/*` — new features
- `fix/*` — bug fixes
- `docs/*` — documentation only

## Commits

- Prefix with type: `feat:`, `fix:`, `docs:`, `chore:`
- Reference JIRA tickets: `[DONUT-123] feat: add custom severity mapping`
- End with attribution:
  ```
  Co-Authored-By: Claude <noreply@anthropic.com>
  ```

## Pull Requests

1. Create PR against `main`
2. Fill in the PR template (see `.github/PULL_REQUEST_TEMPLATE.md`)
3. Request review from `@gdcorp-im/donut` (CODEOWNERS)
4. Squash-merge after approval

## Versioning

After merging:

1. **Patch (v1.0.X)** — bug fixes, doc updates
2. **Minor (v1.X.0)** — new inputs, backwards-compatible features
3. **Major (vX.0.0)** — breaking changes (rare, requires migration plan)

Tag and move floating major tag:

```bash
git tag v1.0.1
git push origin v1.0.1
git tag -f v1 v1.0.1
git push -f origin v1
```

Callers use `@v1` (floating) or `@v1.0.1` (pinned).
