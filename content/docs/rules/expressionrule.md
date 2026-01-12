---
title: "Expression Rule"
weight: 1
---

### Expression Rule Overview

This rule validates GitHub Actions expression syntax (`${{ }}`) throughout workflow files. It performs comprehensive checks including syntax validation, type checking, context availability, and semantic analysis to catch errors before workflows run.

#### Key Features:

- **Syntax Validation**: Parses and validates `${{ }}` expression syntax
- **Type Checking**: Verifies expression types match expected contexts (string, bool, number, object, array)
- **Context Availability**: Ensures contexts (github, env, secrets, matrix, steps, needs, inputs, jobs) are used in valid scopes
- **Semantic Analysis**: Validates property access, function calls, and operators
- **Matrix Type Inference**: Tracks matrix variable types across jobs
- **Steps/Needs Context Tracking**: Validates step outputs and job dependencies

### Security Impact

**Severity: Medium (5/10)**

Invalid expressions can lead to:

1. **Workflow Failures**: Syntax errors cause jobs to fail at runtime
2. **Logic Errors**: Type mismatches can cause unexpected behavior in conditionals
3. **Information Disclosure**: Incorrect context usage might expose unintended data
4. **Security Bypass**: Invalid conditions in security-critical jobs may be evaluated incorrectly
5. **Build Failures**: Expression errors in matrix configurations break parallel builds

### Understanding GitHub Actions Expressions

GitHub Actions uses expressions with `${{ }}` syntax for dynamic values:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Display context
        run: |
          echo "Repository: ${{ github.repository }}"
          echo "Branch: ${{ github.ref_name }}"
          echo "Actor: ${{ github.actor }}"
```

#### Available Contexts

| Context | Description | Availability |
|---------|-------------|--------------|
| `github` | Information about the workflow run | Always |
| `env` | Environment variables | Always |
| `vars` | Repository/organization variables | Always |
| `job` | Current job information | Jobs |
| `jobs` | Outputs from other jobs | `on.workflow_call.outputs` |
| `steps` | Step outputs and status | Steps |
| `runner` | Runner information | Jobs |
| `secrets` | Secret values | Jobs |
| `strategy` | Matrix strategy context | Jobs with matrix |
| `matrix` | Matrix values | Jobs with matrix |
| `needs` | Dependent job outputs | Jobs with `needs:` |
| `inputs` | Workflow inputs | `workflow_call`, `workflow_dispatch` |

### What the Rule Detects

#### 1. Syntax Errors

Invalid expression syntax:

```yaml
# Missing closing bracket
- run: echo "${{ github.actor }"

# Invalid operators
- run: echo "${{ github.actor AND github.repository }}"

# Unclosed string
- run: echo "${{ 'hello }}"
```

#### 2. Type Mismatches

Using wrong types in contexts:

```yaml
# Object cannot be evaluated in template
- run: echo "Event: ${{ github.event }}"

# Array cannot be evaluated in template
- run: echo "Labels: ${{ github.event.pull_request.labels }}"
```

#### 3. Invalid Context Usage

Using contexts outside their valid scope:

```yaml
# 'jobs' context only available in on.workflow_call.outputs
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ jobs.test.outputs.result }}"

# 'matrix' context only available when strategy.matrix is defined
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ matrix.os }}"
```

#### 4. Invalid Property Access

Accessing undefined properties:

```yaml
# Undefined property
- run: echo "${{ github.nonexistent }}"

# Typo in property name
- run: echo "${{ github.repositry }}"

# Wrong context structure
- run: echo "${{ steps.build.output.version }}"  # Should be 'outputs'
```

#### 5. Invalid Function Usage

Incorrect function calls:

```yaml
# Wrong number of arguments
- run: echo "${{ contains(github.ref) }}"

# Invalid argument types
- run: echo "${{ startsWith(123, 'test') }}"

# Undefined function
- run: echo "${{ lowercase(github.actor) }}"
```

### Safe Patterns

#### Pattern 1: Correct Property Access

```yaml
steps:
  - name: Display info
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Ref: ${{ github.ref }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
```

#### Pattern 2: Proper Type Handling

```yaml
steps:
  # Convert object to JSON string
  - run: echo '${{ toJSON(github.event) }}'

  # Access specific properties
  - run: echo "${{ github.event.pull_request.title }}"
```

#### Pattern 3: Valid Conditionals

```yaml
steps:
  - name: Deploy on main
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh

  - name: Skip on PR
    if: github.event_name != 'pull_request'
    run: ./full-build.sh

  - name: Check success
    if: success() && github.actor != 'dependabot[bot]'
    run: ./notify.sh
```

#### Pattern 4: Matrix with Type Safety

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [16, 18, 20]

steps:
  - name: Setup Node
    uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node }}

  - name: Display OS
    run: echo "Running on ${{ matrix.os }}"
```

#### Pattern 5: Steps and Needs References

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - id: version
        run: echo "version=1.0.0" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

### Best Practices

#### 1. Use Proper Type Conversions

```yaml
# Convert objects to JSON for display
- run: echo '${{ toJSON(github.event.inputs) }}'

# Convert to string explicitly
- run: echo "Count: ${{ format('{0}', steps.count.outputs.total) }}"
```

#### 2. Use Functions Correctly

```yaml
steps:
  # contains() with proper types
  - if: contains(github.event.pull_request.labels.*.name, 'urgent')
    run: echo "Urgent PR"

  # startsWith() for branch checks
  - if: startsWith(github.ref, 'refs/tags/')
    run: echo "Tag push"

  # format() for string building
  - run: echo "${{ format('Hello {0}!', github.actor) }}"
```

### Relationship to Other Rules

- **[conditional]({{< ref "conditionalrule.md" >}})**: Validates specific conditional expression patterns
- **[code-injection-critical]({{< ref "codeinjectioncritical.md" >}})**: Detects untrusted input in expressions
- **[envvar-injection-critical]({{< ref "envvarinjectioncritical.md" >}})**: Detects untrusted input written to environment files

### References

- [GitHub Docs: Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [GitHub Docs: Contexts](https://docs.github.com/en/actions/learn-github-actions/contexts)
- [GitHub Docs: Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)

### Configuration

This rule is enabled by default. To disable it:

```bash
sisakulint -ignore expression
```

Disabling this rule is **strongly discouraged** as expression validation catches many common errors before runtime.
