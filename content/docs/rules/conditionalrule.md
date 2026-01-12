---
title: "Conditional Rule"
weight: 1
---

### Conditional Rule Overview

This rule detects a common mistake in GitHub Actions where `if` conditions with multiple `${{ }}` expressions or extra characters always evaluate to `true`. This happens because GitHub Actions treats any non-empty string outside of `${{ }}` as truthy.

#### Key Features:

- **Always-True Detection**: Identifies conditions that will always evaluate to true
- **Multiple Expression Detection**: Catches conditions with multiple `${{ }}` blocks that behave unexpectedly
- **Extra Character Detection**: Warns about conditions with characters outside `${{ }}` brackets

### Security Impact

**Severity: Medium (5/10)**

Incorrect conditional logic can lead to:

1. **Security Bypass**: Critical security checks may be skipped when conditions always evaluate to true
2. **Unintended Deployments**: Deployment jobs might run when they shouldn't
3. **Resource Waste**: Jobs may run unnecessarily, wasting CI/CD resources
4. **Logic Errors**: Workflow behavior becomes unpredictable and hard to debug
5. **Access Control Bypass**: Permission-gated jobs may execute without proper validation

### Understanding GitHub Actions Conditionals

GitHub Actions evaluates `if` conditions in a specific way:

1. **Without `${{ }}`**: The entire string is parsed as an expression
2. **With `${{ }}`**: Only the content inside `${{ }}` is evaluated as an expression

The problem arises when:
- Multiple `${{ }}` blocks are used
- Extra characters exist outside `${{ }}`

In these cases, the string representation of the condition is evaluated, and any non-empty string is truthy.

### Example Vulnerable Workflow

Common conditional mistakes:

```yaml
name: CI Build

on: [push, pull_request]

jobs:
  deploy:
    runs-on: ubuntu-latest
    # PROBLEM: This ALWAYS evaluates to true!
    # The string "${{ github.ref == 'refs/heads/main' }} && ${{ github.event_name == 'push' }}"
    # is a non-empty string, so it's truthy regardless of the actual conditions
    if: ${{ github.ref == 'refs/heads/main' }} && ${{ github.event_name == 'push' }}
    steps:
      - run: echo "Deploying..."

  test:
    runs-on: ubuntu-latest
    # PROBLEM: Extra space/text makes this always true
    if: true ${{ github.actor != 'dependabot[bot]' }}
    steps:
      - run: echo "Testing..."

  release:
    runs-on: ubuntu-latest
    # PROBLEM: Text before ${{ }} makes this always true
    if: Run if ${{ github.ref == 'refs/tags/*' }}
    steps:
      - run: echo "Releasing..."
```

### What the Rule Detects

#### 1. Multiple `${{ }}` Blocks in Conditions

When multiple expression blocks are combined with operators outside the blocks:

```yaml
# Always true - operators are outside ${{ }}
if: ${{ github.ref == 'refs/heads/main' }} && ${{ github.event_name == 'push' }}

# Always true - multiple blocks
if: ${{ condition1 }} || ${{ condition2 }}
```

**Error Output:**

```bash
workflow.yml:10:9: The condition '${{ github.ref == 'refs/heads/main' }} && ${{ github.event_name == 'push' }}' will always evaluate to true. If you intended to use a literal value, please use ${{ true }}. Ensure there are no extra characters within the ${{ }} brackets in conditions. [cond]
```

#### 2. Extra Characters Outside `${{ }}`

Any text outside the expression block causes the condition to always be true:

```yaml
# Always true - "true" text outside brackets
if: true ${{ github.actor != 'bot' }}

# Always true - comment-like text
if: Run if ${{ condition }}

# Always true - trailing space/text
if: ${{ condition }} # this is a comment
```

**Error Output:**

```bash
workflow.yml:10:9: The condition 'true ${{ github.actor != 'bot' }}' will always evaluate to true. If you intended to use a literal value, please use ${{ true }}. Ensure there are no extra characters within the ${{ }} brackets in conditions. [cond]
```

### Safe Patterns

#### Pattern 1: Single Expression Block (Recommended)

Put all logic inside a single `${{ }}`:

```yaml
# Correct: All logic inside one block
if: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}

# Correct: Complex conditions in one block
if: ${{ (github.event_name == 'push' && github.ref == 'refs/heads/main') || github.event_name == 'workflow_dispatch' }}
```

#### Pattern 2: No Expression Brackets

Omit `${{ }}` entirely - GitHub auto-evaluates:

```yaml
# Correct: No brackets needed for simple expressions
if: github.ref == 'refs/heads/main'

# Correct: Complex conditions without brackets
if: github.ref == 'refs/heads/main' && github.event_name == 'push'

# Correct: Functions work without brackets too
if: success() && github.actor != 'dependabot[bot]'
```

#### Pattern 3: Boolean Literals

For explicit true/false:

```yaml
# Correct: Literal true
if: ${{ true }}

# Correct: Literal false
if: ${{ false }}

# Correct: Without brackets
if: true
if: false
```

#### Pattern 4: Using Functions

```yaml
# Correct: Status check functions
if: success()
if: failure()
if: always()
if: cancelled()

# Correct: Combined with conditions
if: ${{ success() && github.ref == 'refs/heads/main' }}

# Without brackets
if: success() && github.ref == 'refs/heads/main'
```

### Auto-Fix Support

sisakulint can automatically fix conditional rule violations by removing unnecessary `${{ }}` wrappers from conditions.

#### How to Apply Auto-Fix

```bash
# Preview changes without modifying files
sisakulint -fix dry-run .github/workflows/

# Apply fixes to files
sisakulint -fix on .github/workflows/
```

### References

- [GitHub Docs: Workflow Syntax - if conditions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idif)
- [GitHub Docs: Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [GitHub Docs: Contexts](https://docs.github.com/en/actions/learn-github-actions/contexts)

### Configuration

This rule is enabled by default. To disable it:

```bash
sisakulint -ignore cond
```

Disabling this rule is **not recommended** as incorrect conditionals can lead to unexpected workflow behavior and potential security bypasses.
