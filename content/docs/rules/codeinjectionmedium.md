---
title: "Code Injection Rule (Medium)"
weight: 13
---

### Code Injection Rule (Medium) Overview

This rule detects code injection vulnerabilities when untrusted input is used directly in shell scripts or JavaScript code within **normal workflow contexts**. While these workflows typically have limited permissions, they can still be exploited to leak information, manipulate builds, or serve as stepping stones for more serious attacks.

#### Key Features:

- **Normal Trigger Detection**: Identifies dangerous patterns in `pull_request`, `push`, `schedule`, and other non-privileged triggers
- **Dual Script Detection**: Analyzes both `run:` scripts and `actions/github-script` for untrusted input
- **Auto-fix Support**: Automatically converts unsafe patterns to use environment variables
- **Build Integrity Protection**: Prevents manipulation of test results, artifacts, and build processes

### Security Impact

**Severity: Medium (6/10)**

Code injection in normal workflows presents moderate security risks:

1. **Information Disclosure**: Leaking environment details, dependencies, or build configurations
2. **Build Manipulation**: Altering test results, coverage reports, or build artifacts
3. **CI/CD Workflow Disruption**: Causing builds to fail or hang
4. **Stepping Stone Attacks**: Gathering information for more targeted attacks

### Normal Workflow Triggers

- **`pull_request`**: Read-only access, no secrets by default
- **`push`**: Triggered only by trusted commits
- **`schedule`**: Time-based triggers with read-only access
- **`workflow_dispatch`**: Manual triggers (trusted users only)

### Example Vulnerable Workflow

```yaml
name: PR Analysis

on:
  pull_request:  # NORMAL: Read-only by default
    types: [opened, synchronize]

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      # MEDIUM VULNERABILITY
      - name: Analyze PR title
        run: |
          echo "Analyzing: ${{ github.event.pull_request.title }}"
```

### Example Output

```bash
$ sisakulint

.github/workflows/pr-analyze.yaml:11:20: code injection (medium): "github.event.pull_request.title" is potentially untrusted. [code-injection-medium]
```

### Auto-fix Support

**Before (Vulnerable):**
```yaml
- name: Test PR
  run: |
    echo "PR: ${{ github.event.pull_request.title }}"
```

**After (Secure):**
```yaml
- name: Test PR
  run: |
    echo "PR: $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

### Common Untrusted Inputs

- `github.event.pull_request.title`
- `github.event.pull_request.body`
- `github.event.pull_request.head.ref`
- `github.event.head_commit.message`
- `github.head_ref`

### Difference from Critical Severity

| Trigger Type | Rule | Risk Level | Permissions |
|--------------|------|------------|-------------|
| `pull_request` | Medium | 6/10 | Read-only |
| `pull_request_target` | Critical | 10/10 | Write + secrets |
| `push` | Medium | 6/10 | Read-only (usually) |
| `workflow_run` | Critical | 10/10 | Elevated privileges |

### Best Practices

1. **Always Use Environment Variables**
2. **Validate Input Even in Limited Contexts**
3. **Sanitize Before Display or Logging**
4. **Limit Output Exposure**

### Related Rules

- **[code-injection-critical]({{< ref "codeinjectioncritical.md" >}})**: Detect severe issues in privileged triggers
- **[envvar-injection-medium]({{< ref "envvarinjectionmedium.md" >}})**: Specialized detection for $GITHUB_ENV writes

### See Also

- [CodeQL: Code Injection (Medium)](https://codeql.github.com/codeql-query-help/actions/actions-code-injection-medium/)
- [GitHub: Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [OWASP: Defense in Depth](https://owasp.org/www-community/Defense_in_depth)

{{< popup_link2 href="https://codeql.github.com/codeql-query-help/actions/actions-code-injection-medium/" >}}

{{< popup_link2 href="https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions" >}}
