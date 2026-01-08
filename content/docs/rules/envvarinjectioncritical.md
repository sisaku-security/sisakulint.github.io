---
title: "Environment Variable Injection Rule (Critical)"
weight: 14
---

### Environment Variable Injection Rule (Critical) Overview

This rule detects environment variable injection vulnerabilities when untrusted input is written to `$GITHUB_ENV` within **privileged workflow contexts**. Privileged workflows have write permissions or access to secrets, making environment variable injection particularly dangerous as it can persist malicious values across workflow steps.

#### Key Features:

- **Privileged Context Detection**: Identifies dangerous patterns in `pull_request_target`, `workflow_run`, `issue_comment`, and other privileged triggers
- **GITHUB_ENV Write Detection**: Analyzes scripts that write to `$GITHUB_ENV` file
- **Auto-fix Support**: Automatically sanitizes inputs using `tr -d '\n'` to prevent newline injection

### Security Impact

**Severity: Critical (9/10)**

Environment variable injection in privileged workflows represents a critical vulnerability:

1. **Environment Pollution**: Attackers can inject arbitrary environment variables affecting subsequent steps
2. **LD_PRELOAD Attacks**: Injection of `LD_PRELOAD` can hijack library loading for code execution
3. **BASH_ENV Attacks**: Malicious `BASH_ENV` can execute code in every shell invocation
4. **Secret Exfiltration**: Injected variables can capture or override security-critical values
5. **Persistence Across Steps**: Unlike in-memory code injection, environment variables persist throughout the job

### Example Vulnerable Workflow

```yaml
name: Process PR Metadata

on:
  pull_request_target:  # PRIVILEGED
    types: [opened, edited]

jobs:
  process:
    runs-on: ubuntu-latest
    steps:
      # CRITICAL VULNERABILITY
      - name: Set PR variables
        run: |
          echo "PR_TITLE=${{ github.event.pull_request.title }}" >> "$GITHUB_ENV"
```

### Attack Scenario

Attacker creates a PR with a crafted title containing newlines:
```
Title: Legit Feature
LD_PRELOAD=/tmp/evil.so
BASH_ENV=/tmp/backdoor.sh
```

The malicious input writes multiple environment variables that affect all subsequent steps.

### Safe Pattern (Using Sanitization)

```yaml
- name: Set PR variables safely
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

### Rule Interactions

You may see both `envvar-injection-critical` and `code-injection-critical` errors on the same line. This is intentional:

- **code-injection-critical**: Moves expressions to `env:` section
- **envvar-injection-critical**: Adds `tr -d '\n'` to $GITHUB_ENV writes

Both auto-fixes work together for complete protection.

### Related Rules

- **[envvar-injection-medium]({{< ref "envvarinjectionmedium.md" >}})**: Detects the same pattern in normal workflows
- **[code-injection-critical]({{< ref "codeinjectioncritical.md" >}})**: Detects direct code injection in privileged contexts

### References

- [CodeQL: Environment Variable Injection (Critical)](https://codeql.github.com/codeql-query-help/actions/actions-envvar-injection-critical/)
- [GitHub Security: Script Injection](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections)

{{< popup_link2 href="https://codeql.github.com/codeql-query-help/actions/actions-envvar-injection-critical/" >}}

{{< popup_link2 href="https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions" >}}
