---
title: "Rules"
weight: 1
---

# sisakulint Rules

sisakulint provides comprehensive security rules for GitHub Actions workflows. Rules are categorized by the security risks they address.

## Security Rules Overview

### Code Injection / Poisoned Pipeline Execution (CICD-SEC-04)

| Rule | Severity | Description |
|------|----------|-------------|
| [code-injection-critical]({{< ref "codeinjectioncritical.md" >}}) | Critical | Detects untrusted input in privileged workflow triggers |
| [code-injection-medium]({{< ref "codeinjectionmedium.md" >}}) | Medium | Detects untrusted input in normal workflow triggers |
| [envvar-injection-critical]({{< ref "envvarinjectioncritical.md" >}}) | Critical | Detects untrusted input written to $GITHUB_ENV in privileged triggers |
| [envvar-injection-medium]({{< ref "envvarinjectionmedium.md" >}}) | Medium | Detects untrusted input written to $GITHUB_ENV in normal triggers |
| [envpath-injection-critical]({{< ref "envpathinjectioncritical.md" >}}) | Critical | Detects untrusted input written to $GITHUB_PATH in privileged triggers |
| [envpath-injection-medium]({{< ref "envpathinjectionmedium.md" >}}) | Medium | Detects untrusted input written to $GITHUB_PATH in normal triggers |
| [untrusted-checkout]({{< ref "untrustedcheckout.md" >}}) | Critical | Detects checkout of untrusted PR code in privileged contexts |
| [improper-access-control]({{< ref "improperaccesscontrol.md" >}}) | Critical | Detects label-based approval bypass vulnerabilities |

### Artifact and Cache Poisoning (CICD-SEC-09)

| Rule | Severity | Description |
|------|----------|-------------|
| [artifact-poisoning-critical]({{< ref "artifactpoisoningcritical.md" >}}) | Critical | Detects artifact poisoning in privileged workflows |
| [artifact-poisoning-medium]({{< ref "artifactpoisoningmedium.md" >}}) | Medium | Detects artifact poisoning in normal workflows |
| [cache-poisoning]({{< ref "cachepoisoningrule.md" >}}) | High | Detects cache poisoning vulnerabilities |
| [cache-poisoning-poisonable-step]({{< ref "cachepoisoningpoisonablesteprule.md" >}}) | High | Detects poisonable steps after unsafe checkout |

### Identity and Access Management (CICD-SEC-02)

| Rule | Severity | Description |
|------|----------|-------------|
| [permissions]({{< ref "permissions.md" >}}) | High | Validates GITHUB_TOKEN permission scopes |
| [secret-exposure]({{< ref "secretexposure.md" >}}) | High | Detects excessive secrets exposure patterns |

### Credential Hygiene (CICD-SEC-06)

| Rule | Severity | Description |
|------|----------|-------------|
| [credentials]({{< ref "CredentialsRule.md" >}}) | High | Detects hardcoded credentials using Rego |

### Third Party Services (CICD-SEC-08)

| Rule | Severity | Description |
|------|----------|-------------|
| [action-list]({{< ref "actionlist.md" >}}) | Medium | Enforces action allowlist/blocklist policies |
| [commit-sha]({{< ref "commitSHARule.md" >}}) | High | Validates commit SHA pinning in actions |

### Workflow Validation

| Rule | Severity | Description |
|------|----------|-------------|
| [id]({{< ref "idRule.md" >}}) | Low | Validates job and step IDs |
| [job-needs]({{< ref "jobneeds.md" >}}) | Low | Validates job dependencies |
| [workflow-call]({{< ref "workflowcall.md" >}}) | Medium | Validates reusable workflow calls |
| [timeout-minutes]({{< ref "timeoutminutesrule.md" >}}) | Medium | Ensures timeout-minutes is set |

### Expression and Syntax Validation

| Rule | Severity | Description |
|------|----------|-------------|
| [expression]({{< ref "expressionrule.md" >}}) | Medium | Validates GitHub Actions expression syntax |
| [conditional]({{< ref "conditionalrule.md" >}}) | Medium | Validates conditional expressions |
| [environment-variable]({{< ref "environmentvariablerule.md" >}}) | Low | Validates environment variable names |
| [deprecated-commands]({{< ref "deprecatedcommandsrule.md" >}}) | High | Detects deprecated workflow commands |

## Auto-Fix Support

The following rules support automatic fixing with `sisakulint -fix on`:

- **timeout-minutes** - Adds default timeout-minutes: 5
- **commit-sha** - Converts action tags to commit SHAs
- **credentials** - Removes hardcoded passwords
- **code-injection-critical/medium** - Moves untrusted expressions to environment variables
- **envvar-injection-critical/medium** - Sanitizes untrusted input before writing to $GITHUB_ENV
- **envpath-injection-critical/medium** - Validates paths with `realpath` before writing to $GITHUB_PATH
- **untrusted-checkout** - Adds explicit ref to checkout in privileged contexts
- **artifact-poisoning-critical/medium** - Adds validation steps for artifact downloads
- **improper-access-control** - Replaces mutable refs with immutable SHAs and changes event types
- **conditional** - Removes unnecessary `${{ }}` wrappers
- **secret-exposure** - Converts bracket notation to dot notation

## OWASP CI/CD Top 10 Mapping

| OWASP Risk | sisakulint Rules |
|------------|------------------|
| CICD-SEC-02 | permissions, secret-exposure |
| CICD-SEC-04 | code-injection-*, envvar-injection-*, envpath-injection-*, untrusted-checkout, improper-access-control |
| CICD-SEC-06 | credentials |
| CICD-SEC-08 | action-list, commit-sha |
| CICD-SEC-09 | artifact-poisoning-*, cache-poisoning-* |
