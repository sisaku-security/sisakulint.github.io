---
title: "Improper Access Control Rule"
weight: 1
---

### Improper Access Control Rule Overview

This rule detects improper access control vulnerabilities in GitHub Actions workflows that use label-based approval mechanisms. This is a **critical security vulnerability** (CVSS 9.3, CWE-285) that allows attackers to bypass label-based approval and execute malicious code.

**Vulnerable Example:**

```yaml
name: PR Build
on:
  pull_request_target:
    types: [opened, synchronize]  # Dangerous: synchronize allows code changes after approval

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        if: contains(github.event.pull_request.labels.*.name, 'safe to test')
        with:
          ref: ${{ github.event.pull_request.head.ref }}  # Mutable reference!
      - run: npm test
```

**Detection Output:**

```bash
vulnerable.yaml:12:9: improper access control: checkout uses label-based approval with 'synchronize' event type and mutable ref. An attacker can modify code after label approval. Fix: 1) Change trigger types from 'synchronize' to 'labeled', 2) Use immutable 'github.event.pull_request.head.sha' instead of mutable 'head.ref'. See https://codeql.github.com/codeql-query-help/actions/actions-improper-access-control/ [improper-access-control]
      12 |      - uses: actions/checkout@v4
```

### Security Background

#### Why is this dangerous?

The vulnerability occurs when three conditions are met:

1. **`pull_request_target` trigger with `synchronize` event type** - Allows the workflow to trigger when new commits are pushed to the PR
2. **Label-based approval check** - Uses labels like "safe to test" to gate execution
3. **Mutable branch reference** - Uses `head.ref` instead of `head.sha`

#### Attack Scenario

```
1. Attacker opens PR with benign code
   └── Workflow does NOT run (no "safe to test" label)

2. Maintainer reviews code and adds "safe to test" label
   └── Workflow runs with benign code

3. Attacker pushes malicious commit to same PR
   └── "synchronize" event triggers workflow
   └── "safe to test" label is still present
   └── Workflow runs with MALICIOUS code!

4. Mutable ref (head.ref) points to the NEW malicious code
   └── Attacker's code executes with access to secrets
```

#### Why `head.ref` vs `head.sha` matters

| Reference | Type | After new push | Security |
|-----------|------|----------------|----------|
| `head.ref` | Branch name | Points to NEW commit | Mutable |
| `head.sha` | Commit SHA | Points to SAME commit | Immutable |

Using `head.ref` (branch name) means the checkout will always get the latest code, even after approval. Using `head.sha` (commit SHA) locks the checkout to the specific commit that was approved.

#### OWASP and CWE Mapping

- **CWE-285:** Improper Authorization
- **OWASP Top 10 CI/CD Security Risks:**
  - **CICD-SEC-4:** Poisoned Pipeline Execution (PPE)

### Safe Patterns

**Safe Pattern 1: Use `labeled` event type only**

```yaml
on:
  pull_request_target:
    types: [labeled]  # Only triggers when label is added - no subsequent pushes

jobs:
  test:
    steps:
      - uses: actions/checkout@v4
        if: contains(github.event.pull_request.labels.*.name, 'safe to test')
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # Immutable!
```

**Safe Pattern 2: Use immutable SHA reference**

```yaml
on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  test:
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # Immutable!
```

**Safe Pattern 3: Use `pull_request` trigger instead**

```yaml
on: pull_request  # No secrets access, safe to run PR code

jobs:
  test:
    steps:
      - uses: actions/checkout@v4  # Safe: no privileged context
```

### Auto-Fix

This rule supports automatic fixing. When you run sisakulint with the `-fix on` flag, it will automatically apply two fixes:

**Fix 1: Replace mutable refs with immutable SHAs**

- `ref: ${{ github.event.pull_request.head.ref }}` becomes `ref: ${{ github.event.pull_request.head.sha }}`
- `ref: ${{ github.head_ref }}` becomes `ref: ${{ github.event.pull_request.head.sha }}`

**Fix 2: Replace `synchronize` with `labeled` in event types**

- `types: [opened, synchronize]` becomes `types: [opened, labeled]`

**Example:**

Before auto-fix:
```yaml
on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  test:
    steps:
      - uses: actions/checkout@v4
        if: contains(github.event.pull_request.labels.*.name, 'safe to test')
        with:
          ref: ${{ github.event.pull_request.head.ref }}
```

After running `sisakulint -fix on`:
```yaml
on:
  pull_request_target:
    types: [opened, labeled]

jobs:
  test:
    steps:
      - uses: actions/checkout@v4
        if: contains(github.event.pull_request.labels.*.name, 'safe to test')
        with:
          ref: ${{ github.event.pull_request.head.sha }}
```

**Note:** After auto-fix, the workflow will only trigger when a label is added (not on subsequent pushes). This is the correct security behavior for label-gated workflows.

### Remediation Steps

When this rule triggers:

1. **Use auto-fix for quick remediation**
   - Run `sisakulint -fix on` to automatically fix the vulnerability
   - Review the changes to ensure they meet your workflow requirements

2. **Change event types to `labeled` only**
   - Remove `synchronize` from the types array
   - Add `labeled` if not already present
   - This ensures the workflow only runs when approval is explicitly granted

3. **Use immutable SHA references**
   - Replace `head.ref` with `head.sha`
   - This locks the checkout to the specific approved commit

4. **Consider using `pull_request` trigger**
   - If you don't need secrets access, use `pull_request` instead
   - This is the safest option for running untrusted PR code

### References

- [GitHub Docs: Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [GitHub Docs: pull_request_target events](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request_target)
- [CodeQL: Improper Access Control](https://codeql.github.com/codeql-query-help/actions/actions-improper-access-control/)
- [GitHub Security Lab: Preventing pwn requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)
- [OWASP CI/CD Top 10](https://owasp.org/www-project-top-10-ci-cd-security-risks/)
