---
title: "Environment Variable Injection Rule (Medium)"
weight: 15
---

### Environment Variable Injection Rule (Medium) Overview

This rule detects environment variable injection vulnerabilities when untrusted input is written to `$GITHUB_ENV` within **normal workflow contexts**. While these workflows have limited permissions compared to privileged contexts, environment variable injection can still lead to security issues and unexpected behavior.

#### Key Features:

- **Normal Trigger Detection**: Identifies patterns in `pull_request`, `push`, `schedule`, and other standard triggers
- **GITHUB_ENV Write Detection**: Analyzes scripts that write to `$GITHUB_ENV` file
- **Auto-fix Support**: Automatically sanitizes inputs using `tr -d '\n'` to prevent newline injection

### Security Impact

**Severity: Medium (5/10)**

Environment variable injection in normal workflows represents a medium-severity vulnerability:

1. **Workflow Logic Manipulation**: Attackers can alter workflow behavior through environment pollution
2. **Data Integrity Issues**: Injected variables can corrupt build outputs or test results
3. **Defense-in-Depth**: Preventing medium-risk patterns stops privilege escalation paths

### Example Vulnerable Workflow

```yaml
name: CI Build

on:
  pull_request:  # Normal trigger with limited permissions
    types: [opened, synchronize]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # MEDIUM VULNERABILITY
      - name: Set build variables
        run: |
          echo "PR_TITLE=${{ github.event.pull_request.title }}" >> "$GITHUB_ENV"
```

### Attack Scenario

Attacker creates a PR with a crafted title:
```
Title: Fix typo
NODE_OPTIONS=--require=/tmp/malicious.js
CI=false
```

This can skip test suites (via `CI=false`) or alter build configurations.

### Safe Pattern

```yaml
- name: Set build variables safely
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    echo "PR_TITLE=$(echo "$PR_TITLE" | tr -d '\n')" >> "$GITHUB_ENV"
```

### Auto-Fix Example

**Before (Vulnerable):**
```yaml
- name: Set variables
  run: |
    echo "TITLE=${{ github.event.pull_request.title }}" >> "$GITHUB_ENV"
```

**After Auto-Fix (Safe):**
```yaml
- name: Set variables
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    echo "TITLE=$(echo "$PR_TITLE" | tr -d '\n')" >> "$GITHUB_ENV"
```

### Difference from Critical Severity

| Aspect | Critical (Privileged) | Medium (Normal) |
|--------|----------------------|-----------------|
| **Permissions** | Write access, secrets | Read-only |
| **Impact** | Repository compromise | Workflow behavior manipulation |
| **Secret Access** | Yes | No |
| **Urgency** | Immediate fix required | Should be fixed |

### Best Practices

1. **Prefer Direct Usage**: Use `env:` blocks instead of `$GITHUB_ENV` when possible
2. **Sanitize Inputs**: Always use `tr -d '\n'` when writing to `$GITHUB_ENV`
3. **Use Heredoc for Multi-line**: Use unique delimiters for multi-line values

### Related Rules

- **[envvar-injection-critical]({{< ref "envvarinjectioncritical.md" >}})**: Higher severity for privileged workflows
- **[code-injection-medium]({{< ref "codeinjectionmedium.md" >}})**: Detects direct code injection in normal contexts
- **[permissions]({{< ref "permissions.md" >}})**: Ensures workflows follow least-privilege principle

### References

- [CodeQL: Environment Variable Injection (Medium)](https://codeql.github.com/codeql-query-help/actions/actions-envvar-injection-medium/)
- [GitHub Security: Script Injection](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections)

{{< popup_link2 href="https://codeql.github.com/codeql-query-help/actions/actions-envvar-injection-medium/" >}}

{{< popup_link2 href="https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions" >}}
