---
title: "Artifact Poisoning Rule (Critical)"
weight: 8
---

### Artifact Poisoning Rule (Critical) Overview

This rule detects unsafe artifact download practices that may allow artifact poisoning attacks. Artifact poisoning occurs when malicious artifacts from untrusted sources overwrite existing files in the runner workspace, potentially leading to code execution in privileged contexts.

#### Key Features:

- **Artifact Download Detection**: Identifies uses of `actions/download-artifact` without proper path isolation
- **Extraction Path Validation**: Ensures artifacts are extracted to safe, isolated locations
- **Auto-fix Support**: Automatically configures safe extraction paths using `${{ runner.temp }}/artifacts`
- **Supply Chain Protection**: Prevents malicious artifacts from compromising the build environment

### Security Impact

**Severity: Critical (9/10)**

Artifact poisoning represents a critical security vulnerability in CI/CD pipelines:

1. **File Overwriting**: Malicious artifacts can replace legitimate files in the workspace
2. **Code Execution**: Overwritten scripts or binaries may be executed by subsequent steps
3. **Credential Theft**: Modified code can exfiltrate secrets or access tokens
4. **Build Contamination**: Compromised builds can propagate malicious code to production

This vulnerability is classified as **CWE-829: Inclusion of Functionality from Untrusted Control Sphere** and aligns with OWASP CI/CD Security Risk **CICD-SEC-4: Poisoned Pipeline Execution (PPE)**.

### Example Vulnerable Workflow

```yaml
name: Deploy Application

on:
  workflow_run:
    workflows: ["Build"]
    types: [completed]

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      # VULNERABLE: No path specified, downloads to current directory
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: application-bundle

      # DANGEROUS: Executes scripts from downloaded artifacts
      - name: Deploy to production
        run: |
          chmod +x ./deploy.sh
          ./deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

### Attack Scenario

**How Artifact Poisoning Works:**

1. **Attacker Creates Malicious PR**: Opens a pull request with seemingly innocent changes
2. **Build Workflow Runs**: CI workflow builds the PR and uploads artifacts
3. **Malicious Artifact Created**: Attacker includes a malicious `deploy.sh` in the artifact
4. **Deploy Workflow Triggered**: Downstream workflow downloads the artifact
5. **Files Overwritten**: Malicious `deploy.sh` overwrites the legitimate deployment script
6. **Code Execution**: Deploy workflow executes the malicious script with production credentials
7. **Compromise**: Attacker gains access to production environment and secrets

### Example Output

```bash
$ sisakulint

.github/workflows/deploy.yaml:12:9: artifact is downloaded without specifying a safe extraction path at step "Download build artifacts". This may allow artifact poisoning where malicious files overwrite existing files. Consider extracting to a temporary folder like '${{ runner.temp }}/artifacts' to prevent overwriting existing files. [artifact-poisoning]
     12 👈|      - name: Download build artifacts
```

### Auto-fix Support

```bash
# Preview changes without applying
sisakulint -fix dry-run

# Apply fixes
sisakulint -fix on
```

After auto-fix, artifacts are extracted to an isolated temporary directory:

```yaml
- name: Download build artifacts
  uses: actions/download-artifact@v4
  with:
    name: application-bundle
    path: ${{ runner.temp }}/artifacts
```

### Best Practices

#### 1. Always Specify Extraction Path

```yaml
- uses: actions/download-artifact@v4
  with:
    name: my-artifact
    path: ${{ runner.temp }}/artifacts
```

#### 2. Treat Artifacts as Untrusted

```yaml
- name: Validate artifact
  run: |
    sha256sum -c ${{ runner.temp }}/artifacts/checksums.txt
```

#### 3. Use Separate Workflows for Privileged Operations

Isolate workflows with production access from untrusted inputs.

### See Also

- [CodeQL: Artifact Poisoning (Critical)](https://codeql.github.com/codeql-query-help/actions/actions-artifact-poisoning-critical/)
- [GitHub: Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [OWASP: CICD-SEC-04 - Poisoned Pipeline Execution](https://owasp.org/www-project-top-10-ci-cd-security-risks/CICD-SEC-04-Poisoned-Pipeline-Execution)

{{< popup_link2 href="https://codeql.github.com/codeql-query-help/actions/actions-artifact-poisoning-critical/" >}}

{{< popup_link2 href="https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions" >}}
