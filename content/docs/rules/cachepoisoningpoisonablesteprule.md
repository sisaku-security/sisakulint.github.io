---
title: "Cache Poisoning Poisonable Step Rule"
weight: 11
---

### Cache Poisoning Poisonable Step Rule Overview

This rule detects potential cache poisoning vulnerabilities when untrusted code is executed after checking out PR head code in privileged workflow contexts. Unlike the `cache-poisoning` rule which focuses on direct cache action usage, this rule focuses on **code execution** that could steal cache tokens or poison cache entries indirectly.

**Vulnerable Example:**

```yaml
name: Vulnerable Workflow
on:
  pull_request_target:
    branches: [main]

permissions: {}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Run tests
        run: ./run_tests.sh  # Executes attacker-controlled code!
```

**Detection Output:**

```bash
workflow.yaml:15:9: cache poisoning risk: executing local script './run_tests.sh' after unsafe checkout in privileged context. Attacker can steal ACTIONS_RUNTIME_TOKEN and poison cache. [cache-poisoning-poisonable-step]
      15 👈|      - name: Run tests
```

### Security Background

#### Why This is Dangerous

This vulnerability combines three dangerous patterns:

1. **Privileged Trigger**: `pull_request_target`, `issue_comment`, or `workflow_run` runs with elevated permissions
2. **Unsafe Checkout**: Checking out untrusted PR code with `ref: ${{ github.event.pull_request.head.sha }}`
3. **Code Execution**: Running local scripts or build commands that execute the checked-out code

#### Attack Scenario

1. Attacker submits malicious PR with modified build scripts
2. `pull_request_target` workflow triggers with default branch permissions
3. Workflow checks out attacker's code
4. Malicious code executes, steals ACTIONS_RUNTIME_TOKEN
5. Future workflow runs restore poisoned cache

### Poisonable Step Patterns

**Local Script Execution:**
```yaml
- run: ./build.sh
- run: bash ./test.sh
```

**Build Commands:**
```yaml
- run: npm install
- run: pip install -r requirements.txt
- run: make
```

**Local Actions:**
```yaml
- uses: ./.github/actions/build
```

### Safe Patterns

#### Using Safe Trigger (`pull_request`)

```yaml
on:
  pull_request:  # Safe: Cache scoped to PR branch
```

#### Safe Checkout (Base Branch)

```yaml
- uses: actions/checkout@v4
  # No 'ref' input: defaults to base branch
```

#### External Commands Only

```yaml
- run: echo "Hello World"  # Safe: doesn't execute local code
```

### Auto-Fix Support

```bash
sisakulint -fix dry-run
sisakulint -fix on
```

The auto-fix removes the `ref` input from checkout steps.

### Related Rules

- **[cache-poisoning]({{< ref "cachepoisoningrule.md" >}})**: Detects cache keys with untrusted input
- **[untrusted-checkout]({{< ref "untrustedcheckout.md" >}})**: Detects checkout of untrusted PR code
- **[code-injection-critical]({{< ref "codeinjectioncritical.md" >}})**: Detects code injection in privileged contexts

### References

- [CodeQL: actions-cache-poisoning-poisonable-step](https://codeql.github.com/codeql-query-help/actions/actions-cache-poisoning-poisonable-step/)
- [GitHub Actions Cache Poisoning](https://adnanthekhan.com/2024/05/06/the-monsters-in-your-build-cache-github-actions-cache-poisoning/)
- [GitHub Security Lab: Preventing Pwn Requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)

{{< popup_link2 href="https://codeql.github.com/codeql-query-help/actions/actions-cache-poisoning-poisonable-step/" >}}

{{< popup_link2 href="https://adnanthekhan.com/2024/05/06/the-monsters-in-your-build-cache-github-actions-cache-poisoning/" >}}
