---
title: "Deprecated Commands Rule"
weight: 1
---

### Deprecated Commands Rule Overview

This rule detects deprecated GitHub Actions workflow commands in `run:` scripts. These commands have been deprecated due to security vulnerabilities and should be replaced with safer alternatives using environment files.

#### Key Features:

- **Deprecated Command Detection**: Identifies `set-output`, `save-state`, `set-env`, and `add-path` commands
- **Migration Guidance**: Provides the recommended replacement for each deprecated command
- **Pattern-Based Detection**: Uses regex to find deprecated commands in shell scripts

### Security Impact

**Severity: High (7/10)**

Deprecated workflow commands were deprecated specifically due to security vulnerabilities:

1. **Command Injection**: The `set-env` and `add-path` commands were vulnerable to injection attacks
2. **Log Poisoning**: Malicious data in logs could inject environment variables or paths
3. **Untrusted Input Exploitation**: Attackers could manipulate workflow behavior through log output
4. **Privilege Escalation**: Injected environment variables could alter subsequent steps' behavior
5. **Supply Chain Risk**: Compromised dependencies could inject malicious values

This aligns with **OWASP CI/CD Security Risk CICD-SEC-04: Poisoned Pipeline Execution (PPE)**.

### Deprecated Commands History

GitHub deprecated these commands in two phases:

#### October 2020: `set-env` and `add-path`

```bash
# Deprecated (vulnerable to injection)
echo "::set-env name=MY_VAR::value"
echo "::add-path::/custom/path"
```

These commands allowed attackers to inject arbitrary environment variables and PATH modifications through workflow logs.

#### October 2022: `set-output` and `save-state`

```bash
# Deprecated
echo "::set-output name=result::value"
echo "::save-state name=data::value"
```

These were deprecated as part of a security hardening initiative and to standardize the output mechanism.

### Example Vulnerable Workflow

Workflow using deprecated commands:

```yaml
name: CI Build

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Deprecated: set-output
      - name: Set version
        id: version
        run: echo "::set-output name=version::1.0.0"

      # Deprecated: save-state
      - name: Save state
        run: echo "::save-state name=build_id::12345"

      # Deprecated: set-env (SECURITY RISK)
      - name: Set environment
        run: echo "::set-env name=MY_VAR::my_value"

      # Deprecated: add-path (SECURITY RISK)
      - name: Add to path
        run: echo "::add-path::/custom/bin"

      - name: Use outputs
        run: echo "Version: ${{ steps.version.outputs.version }}"
```

### Safe Patterns

#### Pattern 1: Setting Step Outputs (Modern)

```yaml
- name: Set version
  id: version
  run: echo "version=1.0.0" >> "$GITHUB_OUTPUT"

- name: Use output
  run: echo "Version: ${{ steps.version.outputs.version }}"
```

#### Pattern 2: Saving State (Modern)

```yaml
- name: Save state
  run: echo "build_id=12345" >> "$GITHUB_STATE"
```

#### Pattern 3: Setting Environment Variables (Modern)

```yaml
- name: Set environment variable
  run: echo "MY_VAR=my_value" >> "$GITHUB_ENV"

- name: Use environment variable
  run: echo "MY_VAR is $MY_VAR"
```

#### Pattern 4: Adding to PATH (Modern)

```yaml
- name: Add to PATH
  run: echo "/custom/bin" >> "$GITHUB_PATH"

- name: Use new path
  run: custom-command  # Now available in PATH
```

#### Pattern 5: Multi-line Outputs

For values containing newlines or special characters:

```yaml
- name: Set multi-line output
  id: changelog
  run: |
    {
      echo 'changelog<<EOF'
      cat CHANGELOG.md
      echo 'EOF'
    } >> "$GITHUB_OUTPUT"

- name: Display changelog
  run: echo "${{ steps.changelog.outputs.changelog }}"
```

### Migration Guide

| Deprecated Command | Modern Replacement |
|-------------------|-------------------|
| `echo "::set-output name=NAME::VALUE"` | `echo "NAME=VALUE" >> "$GITHUB_OUTPUT"` |
| `echo "::save-state name=NAME::VALUE"` | `echo "NAME=VALUE" >> "$GITHUB_STATE"` |
| `echo "::set-env name=NAME::VALUE"` | `echo "NAME=VALUE" >> "$GITHUB_ENV"` |
| `echo "::add-path::PATH"` | `echo "PATH" >> "$GITHUB_PATH"` |

### References

- [GitHub Blog: set-env and add-path Deprecation (Oct 2020)](https://github.blog/changelog/2020-10-01-github-actions-deprecating-set-env-and-add-path-commands/)
- [GitHub Blog: save-state and set-output Deprecation (Oct 2022)](https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/)
- [GitHub Docs: Workflow Commands](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions)
- [GitHub Docs: Setting Outputs](https://docs.github.com/en/actions/using-jobs/defining-outputs-for-jobs)

### Configuration

This rule is enabled by default. To disable it:

```bash
sisakulint -ignore deprecated-commands
```

Disabling this rule is **strongly discouraged** as deprecated commands represent security vulnerabilities and will eventually stop working.
