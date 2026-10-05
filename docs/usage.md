# Usage Guide

Reusable workflow that gates pull requests on newly-introduced vulnerable dependencies (via GitHub's dependency-review-action) and Semgrep findings (via Dodge SARIF uploads or CLI fallback).

## Quick Start

```yaml
name: PR Security Gate
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: read
  pull-requests: write
  security-events: read
  checks: read
  id-token: write

concurrency:
  group: pr-security-gate-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  gate:
    uses: gdcorp-actions/gd-pr-gate/.github/workflows/gate.yml@v1
    with:
      runner_group: 7219-mscicdrunners
```

## Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `runner_group` | string | no | `7219-mscicdrunners` | Self-hosted runner group for all jobs |
| `dep_fail_severity` | string | no | `high` | Minimum vulnerability severity to fail the check (`low`, `moderate`, `high`, `critical`) |
| `semgrep_fail_severity` | string | no | `error` | Minimum Semgrep severity to fail the check (`note`, `warning`, `error`) |
| `block` | boolean | no | `true` | Whether to fail the check when thresholds are exceeded (set `false` for dry-run) |
| `dodge_wait_minutes` | number | no | `15` | Minutes to wait for `semgrep-dodge` check before falling back to CLI |
| `semgrep_fallback_config` | string | no | `p/default` | Semgrep config to use when falling back to CLI |

## Permissions

The workflow requires these permissions on the caller:

```yaml
permissions:
  contents: read         # checkout code
  pull-requests: write   # create/update comments
  security-events: read  # code-scanning API (Semgrep alerts)
  checks: read           # poll for semgrep-dodge check status
  id-token: write        # JFrog OIDC (CLI fallback)
```

## Examples

### Custom Thresholds

Block on moderate+ deps and warning+ Semgrep:

```yaml
jobs:
  gate:
    uses: gdcorp-actions/gd-pr-gate/.github/workflows/gate.yml@v1
    with:
      runner_group: 7219-mscicdrunners2
      dep_fail_severity: moderate
      semgrep_fail_severity: warning
```

### Dry-Run Mode

Report without blocking:

```yaml
jobs:
  gate:
    uses: gdcorp-actions/gd-pr-gate/.github/workflows/gate.yml@v1
    with:
      runner_group: 7219-mscicdrunners
      block: false
```

### Faster Dodge Timeout

If Dodge consistently completes in <5min:

```yaml
jobs:
  gate:
    uses: gdcorp-actions/gd-pr-gate/.github/workflows/gate.yml@v1
    with:
      runner_group: 7219-mscicdrunners
      dodge_wait_minutes: 5
```

## Making the Check Required

After verifying the gate works on a few PRs, make the `gate / report` check required in branch protection:

**Via gh CLI:**
```bash
gh api -X PATCH repos/{owner}/{repo}/branches/main/protection \
  -F "required_status_checks[checks][][context]=gate / report"
```

**Via UI:**
1. Repo **Settings → Branches → main** protection rule
2. **Require status checks to pass before merging**
3. Add `gate / report`

## Pre-requisites

- ✅ GitHub Advanced Security enabled
- ✅ Dependabot alerts enabled
- ✅ Dependency graph enabled
- ✅ gd-dodge app installed (org-wide)
- ✅ Self-hosted runners with network access to `semgrep.dev` (for CLI fallback)

## Verification

Before deploying:

```bash
# Confirm Dodge uploads to code-scanning API
gh api repos/{owner}/{repo}/code-scanning/analyses?tool_name=Semgrep%20OSS | jq '.[].ref'

# Confirm dependency graph is enabled
gh api repos/{owner}/{repo}/dependency-graph/compare/main...main
```

After deploying, test with throwaway PRs:
1. **Clean PR** → expect ✅ comment, check passes
2. **Add `lodash@4.17.11`** (high CVE) → expect ❌ deps table, check fails
3. **Add `eval(req.query.x)`** → expect ❌ Semgrep table, check fails
4. **Fix (2)** → expect comment updates to "✅ resolved"

See [how-it-works.md](how-it-works.md) for detailed behavior.
