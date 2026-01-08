---
title: "Action List Rule"
weight: 7
---

### Action List Rule Overview

This rule enforces a whitelist of allowed GitHub Actions in your workflows. It helps prevent the use of unauthorized or potentially malicious third-party actions, which is a critical security practice for CI/CD pipelines.

**IMPORTANT**: This rule **only activates when `action-list:` is defined in `.github/sisakulint.yaml`**. Without this configuration, all actions are permitted by default.

#### Key Features:

- **Opt-in Activation**: Rule only runs when `action-list:` is configured in `.github/sisakulint.yaml`
- **Whitelist Enforcement**: Only actions matching configured patterns are allowed
- **Wildcard Support**: Use `*` wildcards to match multiple versions (e.g., `actions/checkout@*`)
- **Auto-generation**: Automatically generate whitelist from existing workflows

### Generating Action List Configuration (First-Time Setup)

sisakulint provides a convenient command to automatically generate an action list configuration from your existing workflow files:

```bash
sisakulint -generate-action-list
```

This command will:
1. Scan all workflow files in `.github/workflows/`
2. Extract all action references (e.g., `actions/checkout@v4`)
3. Normalize them to patterns with wildcards (e.g., `actions/checkout@*`)
4. Generate or update `.github/sisakulint.yaml` with the action list

#### Generated Configuration Example:

```yaml
# Configuration file for sisakulint
# Auto-generated action list from existing workflow files

action-list:
  - actions/checkout@*
  - actions/setup-go@*
  - docker/build-push-action@*
  - golangci/golangci-lint-action@*
```

### Pattern Matching

The action list supports flexible pattern matching with wildcards:

- **Version wildcards**: `actions/checkout@*` matches any version
- **Exact matches**: `actions/checkout@v4` matches only v4
- **Local paths**: `./local-action@v1` (preserved as-is)
- **Docker images**: `docker://alpine:latest` (preserved as-is)

### Example Workflow

```yaml
name: CI
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
      - uses: unauthorized/suspicious-action@v1  # Will be flagged
```

### Example Output

```bash
$ sisakulint

.github/workflows/ci.yaml:12:9: action 'unauthorized/suspicious-action@v1' is not in the whitelist in step 'Run Suspicious Action' [action-list]
       12 👈|      - uses: unauthorized/suspicious-action@v1
```

**Note**: This rule does not support auto-fixing. You must manually review and update workflow files to use only approved actions from your whitelist.

### Feature Background

#### Real-World Supply Chain Attack: tj-actions Breach

This rule was created in response to recommendations from the **Black Hat USA 2025 presentation** on the tj-actions supply chain breach:

- **tj-actions/changed-files** was compromised using "imposter commits"
- Attackers injected malicious code to exfiltrate CI/CD secrets
- Traditional security tools failed to detect the attack

**Why Action Allowlist Matters**

With over 25,000 actions available in the GitHub Marketplace, an allowlist approach ensures:
- Only pre-approved, vetted actions can be used
- New or compromised actions are blocked by default
- Organizations maintain control over their CI/CD supply chain

This aligns with OWASP CI/CD Security Risk **CICD-SEC-8: Ungoverned Usage of 3rd Party Services**.

### Configuration Best Practices

1. **Start with generation**: Use `-generate-action-list` to create an initial whitelist
2. **Review and refine**: Manually review the generated list and remove any suspicious actions
3. **Use wildcards wisely**: Balance security with maintenance overhead
4. **Regular updates**: Periodically regenerate and review your action list

### See Also

- [Black Hat USA 2025: tj-actions Supply Chain Breach Analysis](https://www.stepsecurity.io/blog/when-changed-files-changed-everything-our-black-hat-2025-presentation-on-the-tj-actions-supply-chain-breach)
- [OWASP Top 10 CI/CD Security Risks: CICD-SEC-08](https://owasp.org/www-project-top-10-ci-cd-security-risks/CICD-SEC-08-Ungoverned-Usage-of-3rd-Party-Services)

{{< popup_link2 href="https://www.stepsecurity.io/blog/when-changed-files-changed-everything-our-black-hat-2025-presentation-on-the-tj-actions-supply-chain-breach" >}}

{{< popup_link2 href="https://owasp.org/www-project-top-10-ci-cd-security-risks/CICD-SEC-08-Ungoverned-Usage-of-3rd-Party-Services" >}}
