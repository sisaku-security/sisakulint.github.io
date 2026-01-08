---
title: "Timeout Minutes Rule"
weight: 5
---

### Timeout Minutes Rule Overview

This rule enforces the `timeout-minutes` attribute for all jobs in GitHub Actions workflows. Without explicit timeouts, jobs can run indefinitely, consuming CI/CD resources and potentially being exploited for malicious purposes.

**Invalid Example:**

```yaml
name: CI
on: [push, pull_request]

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    # Missing timeout-minutes
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  docker:
    name: Build Docker
    runs-on: ubuntu-latest
    # Missing timeout-minutes
    steps:
      - uses: actions/checkout@v4
      - run: docker build .
```

**Detection Output:**

```bash
CI.yaml:5:3: timeout-minutes is not set for job lint; see https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idtimeout-minutes for more details. [missing-timeout-minutes]
      5 👈|  lint:

CI.yaml:13:3: timeout-minutes is not set for job docker; see https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idtimeout-minutes for more details. [missing-timeout-minutes]
      13 👈|  docker:
```

### Rule Background

#### Why Timeout Configuration Matters

Jobs without explicit timeouts pose several risks:

1. **Resource Exhaustion**: Long-running jobs consume compute minutes and can exhaust CI/CD quotas
2. **Denial of Service**: Malicious PRs could intentionally create infinite loops
3. **C2 Attack Vector**: Compromised workflows could be used as command-and-control infrastructure
4. **Cost Overruns**: Billable minutes accumulate when jobs hang indefinitely
5. **Developer Friction**: Stuck jobs block CI/CD pipelines and delay releases

#### Default Behavior

GitHub Actions has a default timeout of **360 minutes (6 hours)** per job. This is often excessive for most workflows and should be explicitly reduced.

### Auto-Fix Support

The timeout-minutes rule supports auto-fixing by adding a default timeout:

```bash
# Preview changes without applying
sisakulint -fix dry-run

# Apply fixes
sisakulint -fix on
```

**Before (Missing Timeout):**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

**After Auto-Fix:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 5  # Added by sisakulint
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

**Note:** The auto-fix adds a default timeout of 5 minutes. Review and adjust this value based on your job's actual requirements.

### Valid Patterns

#### Pattern 1: Simple Timeout

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

#### Pattern 2: Different Timeouts per Job

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5  # Quick job

  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30  # Longer job

  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 15
```

### Recommended Timeout Values

| Job Type | Recommended Timeout |
|----------|---------------------|
| Linting | 5-10 minutes |
| Unit Tests | 10-20 minutes |
| Integration Tests | 20-45 minutes |
| Build (Simple) | 10-15 minutes |
| Build (Complex) | 20-30 minutes |
| Docker Build | 15-30 minutes |
| Deployment | 10-20 minutes |

### Best Practices

1. **Set Realistic Timeouts**: Choose timeouts based on typical job duration plus buffer
2. **Different Timeouts for Different Jobs**: Match timeout to job complexity
3. **Consider Matrix Jobs**: Matrix jobs may need longer timeouts
4. **Self-Hosted Runners**: Self-hosted runners may need adjusted timeouts

### Related Rules

- **[permissions]({{< ref "permissions.md" >}})**: Limits job permissions
- **[commit-sha]({{< ref "commitSHARule.md" >}})**: Pins actions for security

### References

- [GitHub Docs: timeout-minutes](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idtimeout-minutes)
- [GitHub Docs: Usage Limits](https://docs.github.com/en/actions/learn-github-actions/usage-limits-billing-and-administration)

{{< popup_link2 href="https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idtimeout-minutes" >}}

{{< popup_link2 href="https://docs.github.com/en/actions/learn-github-actions/usage-limits-billing-and-administration" >}}

### Configuration

This rule is enabled by default. To disable it:

```bash
sisakulint -ignore missing-timeout-minutes
```

However, disabling this rule is **not recommended** as explicit timeouts are an important security and resource management practice.
