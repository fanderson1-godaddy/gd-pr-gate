# gd-pr-gate

GitHub Actions reusable workflow that gates PRs on newly-introduced vulnerable dependencies and Semgrep findings.

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

jobs:
  gate:
    uses: gdcorp-actions/gd-pr-gate/.github/workflows/gate.yml@v1
    with:
      runner_group: your-runner-group
```

See [docs/usage.md](docs/usage.md) for full documentation.

## Features

- ✅ Dependency scanning via GitHub's dependency-review-action
- ✅ Semgrep scanning via Dodge SARIF (with CLI fallback)
- ✅ Sticky PR comment with findings tables
- ✅ Configurable thresholds (default: high/critical deps, ERROR Semgrep)
- ✅ Dry-run mode

## Documentation

- [Usage Guide](docs/usage.md) - inputs, permissions, examples
- [How It Works](docs/how-it-works.md) - flow diagram, decision logic
- [Troubleshooting](docs/troubleshooting.md) - common issues

## License

Proprietary - GoDaddy Internal Use Only

