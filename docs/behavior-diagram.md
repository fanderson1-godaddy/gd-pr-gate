# PR Security Gate Workflow Behavior

## Flow Diagram

```
PR opened/updated
       ↓
┌──────────────────────────────────────────────┐
│  Job 1: deps                                 │
│  - Run dependency-review-action              │
│  - Compare PR manifests vs base             │
│  - Output: vulnerable_changes JSON           │
└──────────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────────────┐
│  Job 2: semgrep                              │
│                                              │
│  ┌─────────────────────────────────────┐   │
│  │ Poll for semgrep-dodge check        │   │
│  │ (30s interval, up to 15min)         │   │
│  └─────────────────────────────────────┘   │
│         ↓                  ↓                │
│    Completed           Timeout/Missing      │
│         ↓                  ↓                │
│  ┌──────────────┐   ┌──────────────────┐  │
│  │ Read Dodge   │   │ Setup JFrog OIDC │  │
│  │ SARIF via    │   │ Install semgrep  │  │
│  │ code-scanning│   │ Run CLI scan     │  │
│  │ API          │   │ (baseline diff)  │  │
│  └──────────────┘   └──────────────────┘  │
│         ↓                  ↓                │
│  New = PR alerts - base alerts             │
│  Output: findings JSON + source            │
└──────────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────────────┐
│  Job 3: report (needs: deps, semgrep)        │
│                                              │
│  1. Filter findings by threshold             │
│     - Deps: high/critical                   │
│     - Semgrep: error                        │
│                                              │
│  2. Build markdown comment                   │
│     - Status emoji (✅/❌)                   │
│     - Deps table (pkg, ver, advisory, fix)  │
│     - Semgrep table (rule, file:line, msg)  │
│     - How to fix footer                      │
│                                              │
│  3. Upsert PR comment (find by marker)       │
│     (skip if Dependabot PR)                 │
│                                              │
│  4. Write to step summary                    │
│                                              │
│  5. Exit 1 if block=true and blocking found  │
│     ↓                                        │
│  Check status: passed/failed                 │
└──────────────────────────────────────────────┘
```

## Decision Points

### Should the Check Fail?

```
blocking_deps = deps where severity >= dep_fail_severity (default: high)
blocking_semgrep = semgrep where severity >= semgrep_fail_severity (default: error)
should_fail = block_enabled AND (blocking_deps > 0 OR blocking_semgrep > 0)
```

**Default thresholds:**
- `dep_fail_severity: high` → blocks on high + critical
- `semgrep_fail_severity: error` → blocks on ERROR only
- `block: true` → actually fails the check

**With current alert counts (Oct 2026):**
- No repo has high/critical Dependabot alerts → no immediate blocking
- Semgrep ERROR findings vary by repo

### Semgrep Source Selection

```
if semgrep-dodge completes within dodge_wait_minutes:
    source = "dodge"
    findings = code-scanning API diff (PR alerts - base alerts)
    ruleset = godaddy.* (internal)
else:
    source = "cli"
    findings = semgrep scan --baseline-commit output
    ruleset = p/default (public)
    comment shows "⚠️ Fallback ruleset used"
```

**Dodge reliability:**
- Target: <5min per scan
- If consistently >15min: increase `dodge_wait_minutes` or accept CLI fallback

## Example Scenarios

### Scenario 1: Clean PR
- **Deps:** 0 vulnerable changes
- **Semgrep:** 0 new findings
- **Comment:** "✅ No security issues found."
- **Check:** ✅ Passed

### Scenario 2: Low-Severity Dep
- **Deps:** 1 `lodash` low severity
- **Semgrep:** 0 new findings
- **Comment:** Deps table with 1 row (no 🚫 marker)
- **Check:** ✅ Passed (below `high` threshold)

### Scenario 3: High-Severity Dep
- **Deps:** 1 `lodash@4.17.11` high severity
- **Semgrep:** 0 new findings
- **Comment:** "🚫 **This PR introduces security issues**" + Deps table with 🚫 marker
- **Check:** ❌ Failed (meets `high` threshold)

### Scenario 4: Semgrep WARNING
- **Deps:** 0 vulnerable changes
- **Semgrep:** 1 `javascript.lang.security.audit.eval-detected` WARNING
- **Comment:** Semgrep table with 1 row (no 🚫 marker)
- **Check:** ✅ Passed (below `error` threshold)

### Scenario 5: Semgrep ERROR
- **Deps:** 0 vulnerable changes
- **Semgrep:** 1 `godaddy.nodejs.security.hardcoded-secret` ERROR
- **Comment:** "🚫 **This PR introduces security issues**" + Semgrep table with 🚫 marker
- **Check:** ❌ Failed (meets `error` threshold)

### Scenario 6: Both High + ERROR
- **Deps:** 1 `axios` high severity
- **Semgrep:** 1 `sql-injection` ERROR
- **Comment:** Both tables, both with 🚫 markers
- **Check:** ❌ Failed (both thresholds met)

### Scenario 7: Dependabot PR
- **Actor:** `dependabot[bot]`
- **Deps:** 1 high severity (the fix)
- **Comment:** Skipped (read-only token)
- **Step Summary:** Markdown report visible in workflow run
- **Check:** ✅ Passed (Dependabot PRs upgrade *away* from vulns, so the diff shows the vuln being removed, not added)

## Comment Updates

The comment is **sticky** — it uses a marker `<!-- pr-security-gate -->` to find the existing comment:

```
Push 1: [vulnerable dep added]
  → New comment created with ❌ status

Push 2: [vuln fixed]
  → Same comment updated to ✅ "No security issues found"

Push 3: [new Semgrep finding]
  → Same comment updated to ❌ with Semgrep table
```

No comment spam; one comment per PR, always at the same position in the timeline.

## Permissions Required

```yaml
permissions:
  contents: read         # checkout code
  pull-requests: write   # create/update comments
  security-events: read  # code-scanning API (Semgrep alerts)
  checks: read           # poll for semgrep-dodge check status
  id-token: write        # JFrog OIDC (CLI fallback)
```

**What each permission does:**
- `contents: read` — `actions/checkout` needs this to clone the repo
- `pull-requests: write` — `github.rest.issues.createComment` / `updateComment` for the sticky comment
- `security-events: read` — `github.rest.codeScanning.listAlertsForRepo` to read Semgrep SARIF
- `checks: read` — `github.rest.checks.listForRef` to poll for `semgrep-dodge` check run
- `id-token: write` — `jfrog/setup-jfrog-cli` OIDC flow to authenticate to Artifactory PyPI

## API Rate Limits

Per PR run:
- **Dodge polling:** up to 30 API calls (`dodge_wait_minutes * 2`, default 30)
- **Code-scanning alerts:** 2 calls (PR ref + base ref)
- **Comment upsert:** 2 calls (list comments, then create or update)
- **Check listForRef:** 1 call per poll

**Total per PR:** ~35 API calls if Dodge takes the full 15 minutes. Most runs: ~5 calls.

## Performance

**Typical run time:**
- `deps` job: 30-60s (lockfile parsing + GitHub Advisory DB lookup)
- `semgrep` job: 5-15min (waiting for Dodge) or 2-5min (CLI fallback)
- `report` job: 10-20s (comment upsert + markdown generation)

**Total:** 6-16 minutes wall-clock, dominated by Dodge wait.

**Optimization:**
- Reduce `dodge_wait_minutes` to 5-10 if Dodge consistently completes faster
- Use `concurrency` to cancel in-progress runs when a new push arrives

## Dry-Run Mode

Set `block: false` to run the gate without ever failing:

```yaml
jobs:
  gate:
    uses: gdcorp-actions/gd-pr-gate/.github/workflows/gate.yml@v1
    with:
      runner_group: 7219-mscicdrunners
      block: false
```

**Effect:**
- Comment still posted
- Step summary still written
- Check always shows ✅ Passed (even if blocking issues found)

**Use cases:**
- Initial rollout (observe before enforcing)
- Repos with many existing alerts (surface new issues without blocking existing work)
- Shadow mode (compare gate results against manual review)

---

Generated 2026-10-05 by Claude Code
