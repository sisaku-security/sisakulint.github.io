---
title: "Permissions Rule"
weight: 4
---

### Permissions Rule Overview

This rule enforces the principle of least privilege by validating permission settings in GitHub Actions workflows. It ensures that workflows explicitly define appropriate permission scopes and use only valid permission values, reducing the attack surface and preventing accidental privilege escalation.

#### Key Features:

- **Top-Level Permissions Validation**: Ensures workflow-level permissions use only `read-all`, `write-all`, or `none`
- **Job-Level Permissions Validation**: Validates job-specific permission scopes
- **Scope Validation**: Checks that only valid permission scopes are used
- **Value Validation**: Ensures permission values are limited to `read`, `write`, or `none`

### Security Impact

**Severity: High (7/10)**

Misconfigured permissions in GitHub Actions workflows can lead to serious security issues:

1. **Privilege Escalation**: Overly broad permissions grant unnecessary access to repository resources
2. **Token Abuse**: Workflows with `write-all` permissions can modify code, releases, and deployments
3. **Secret Exposure Risk**: Excessive permissions increase the attack surface for credential theft
4. **Supply Chain Attacks**: Write access to packages or releases can enable supply chain compromise

This aligns with **OWASP CI/CD Security Risk CICD-SEC-02: Inadequate Identity and Access Management**.

### Understanding GitHub Actions Permissions

#### Permission Scopes

Available permission scopes include:

| Scope | Controls Access To |
|-------|-------------------|
| `actions` | GitHub Actions runs and artifacts |
| `checks` | Check runs and check suites |
| `contents` | Repository contents (code, releases) |
| `deployments` | Deployment statuses |
| `discussions` | GitHub Discussions |
| `id-token` | OIDC token generation |
| `issues` | Issues and issue comments |
| `packages` | GitHub Packages |
| `pages` | GitHub Pages |
| `pull-requests` | Pull requests and comments |
| `repository-projects` | Classic repository projects |
| `security-events` | Code scanning alerts |
| `statuses` | Commit statuses |

#### Permission Values

Each scope accepts three values:

- **`read`**: Read-only access (recommended default)
- **`write`**: Read and write access (use sparingly)
- **`none`**: No access (explicitly deny)

### Example Vulnerable Workflow

```yaml
name: CI Build

on: [push, pull_request]

# PROBLEM: Using invalid permission value
permissions: write  # Invalid - should be "write-all", "read-all", or "none"

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      # PROBLEM: Invalid scope name
      check: write  # Should be "checks" not "check"

      # PROBLEM: Invalid permission value
      issues: readable  # Should be "read", "write", or "none"

    steps:
      - uses: actions/checkout@v4
      - run: npm test
```

### What the Rule Detects

```bash
$ sisakulint .github/workflows/ci.yml

.github/workflows/ci.yml:4:14: "write" is invalid for permission for all the scopes. [permissions]
     4 👈|permissions: write

.github/workflows/ci.yml:11:7: unknown permission scope "check". all available permission scopes are "actions", "checks", "contents", "deployments", "discussions", "id-token", "issues", "packages", "pages", "pull-requests", "repository-projects", "security-events", "statuses" [permissions]
     11 👈|      check: write

.github/workflows/ci.yml:13:15: The value "readable" is not a valid permission for the scope "issues". Only 'read', 'write', or 'none' are acceptable values. [permissions]
     13 👈|      issues: readable
```

### Safe Patterns

#### Pattern 1: Minimal Permissions (Recommended)

```yaml
name: CI Build

on: [push, pull_request]

permissions:
  contents: read      # Can checkout code
  pull-requests: read # Can read PR metadata

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

#### Pattern 2: Job-Level Permissions

```yaml
name: Build and Release

on:
  push:
    tags: ['v*']

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build

  release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write    # Can create releases
      packages: write    # Can publish packages
    steps:
      - uses: actions/checkout@v4
      - uses: actions/create-release@v1
```

### Best Practices

1. **Start with Minimal Permissions**
2. **Use Job-Level Permissions** to isolate privileged operations
3. **Avoid `write-all`** except when absolutely necessary
4. **Document Why Write Access is Needed**
5. **Review Inherited Permissions** - always be explicit

### Related Rules

- **[envvar-injection-critical]({{< ref "envvarinjectioncritical.md" >}})**: Critical severity because privileged workflows have write permissions
- **[code-injection-critical]({{< ref "codeinjectioncritical.md" >}})**: More dangerous in workflows with elevated permissions
- **[untrustedcheckout]({{< ref "untrustedcheckout.md" >}})**: Checking out untrusted code with write permissions is especially risky

### References

- [GitHub Docs: Automatic Token Authentication](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)
- [GitHub Docs: Permissions for GITHUB_TOKEN](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)
- [GitHub Security: Hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

{{< popup_link2 href="https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token" >}}

### Configuration

This rule is enabled by default. To disable it:

```bash
sisakulint -ignore permissions
```

However, disabling this rule is **strongly discouraged** as proper permission configuration is fundamental to GitHub Actions security.
