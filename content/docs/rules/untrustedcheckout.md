---
title: "Untrusted Checkout Rule"
weight: 16
---

### Untrusted Checkout Rule Overview

This rule detects when workflows with privileged triggers check out untrusted code from pull requests. This is a **critical security vulnerability** (CVSS 9.3) that allows attackers to exfiltrate secrets or compromise the repository.

**Vulnerable Example:**

```yaml
name: PR Build
on: pull_request_target  # Dangerous: Runs in base repo context with secrets access

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # Checking out untrusted PR code
      - run: npm install  # Malicious code can access ${{ secrets.NPM_TOKEN }}
```

**Detection Output:**

```bash
vulnerable.yaml:9:16: checking out untrusted code from pull request in workflow with privileged trigger 'pull_request_target'. This allows potentially malicious code from external contributors to execute with access to repository secrets. [untrusted-checkout]
      9 👈|          ref: ${{ github.event.pull_request.head.sha }}
```

### Security Background

#### Why is this dangerous?

| Trigger | Context | Secrets Access | Write Permissions |
|---------|---------|----------------|-------------------|
| `pull_request` | PR context (fork) | No | No (read-only) |
| `pull_request_target` | Base repo context | Yes | Yes |
| `issue_comment` | Base repo context | Yes | Yes |
| `workflow_run` | Base repo context | Yes | Yes |

**The Vulnerability:** When using `pull_request_target`, `issue_comment`, `workflow_run`, or `workflow_call` triggers and checking out code from the pull request HEAD, external attackers can:

1. **Exfiltrate Secrets**
2. **Modify Repository**
3. **Compromise CI/CD**
4. **Supply Chain Attack**

### Dangerous Triggers Detected

1. **`pull_request_target`**: Runs in base repository context with secrets access
2. **`issue_comment`**: Triggered by comments on PRs from external contributors
3. **`workflow_run`**: Triggered after another workflow completes
4. **`workflow_call`**: Inherits security context of the calling workflow

### Untrusted Ref Patterns

- `${{ github.event.pull_request.head.sha }}`
- `${{ github.event.pull_request.head.ref }}`
- Any expression containing `github.event.pull_request.head.*`

### Safe Patterns

**Safe Alternative 1: Use `pull_request` trigger**

```yaml
on: pull_request  # No secrets access, read-only permissions
```

**Safe Alternative 2: Don't checkout PR code**

```yaml
on: pull_request_target
jobs:
  label:
    steps:
      # No checkout - only use GitHub API
      - uses: actions/github-script@v7
```

**Safe Alternative 3: Two-workflow pattern**

Separate untrusted execution from privileged operations.

### Auto-Fix

The rule supports automatic fixing:

**Before auto-fix:**
```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ github.event.pull_request.head.sha }}
```

**After auto-fix:**
```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ github.sha }}
```

```bash
sisakulint -fix on
```

### Remediation Steps

1. **Use auto-fix for quick remediation**
2. **Assess if you need privileged access**
3. **Use the two-workflow pattern**
4. **Avoid checking out PR code**
5. **Review existing workflows**

### References

- [GitHub: Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [CodeQL: Untrusted Checkout (Critical)](https://codeql.github.com/codeql-query-help/actions/actions-untrusted-checkout-critical/)
- [GitHub Security Lab: Preventing Pwn Requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)
- [OWASP CI/CD Top 10](https://owasp.org/www-project-top-10-ci-cd-security-risks/)

{{< popup_link2 href="https://codeql.github.com/codeql-query-help/actions/actions-untrusted-checkout-critical/" >}}

{{< popup_link2 href="https://securitylab.github.com/research/github-actions-preventing-pwn-requests/" >}}

{{< popup_link2 href="https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions" >}}
