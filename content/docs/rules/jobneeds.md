---
title: "Job Needs Rule"
weight: 1
---

### Job Needs Rule Overview

This rule validates job dependencies in GitHub Actions workflows by checking the `needs:` configuration. It detects duplicate dependencies, references to undefined jobs, and cyclic dependency chains that would cause workflow failures.

#### Key Features:

- **Duplicate Detection**: Identifies repeated job IDs in the `needs:` section
- **Undefined Job Detection**: Catches references to jobs that don't exist in the workflow
- **Cyclic Dependency Detection**: Finds circular dependency chains using DAG (Directed Acyclic Graph) analysis
- **Job ID Collision Detection**: Warns when the same job ID is defined multiple times

### Security Impact

**Severity: Low (3/10)**

While not a direct security issue, invalid job dependencies can lead to:

1. **Workflow Failures**: Workflows with invalid dependencies won't run, potentially blocking CI/CD pipelines
2. **Skipped Security Checks**: If security-related jobs have broken dependencies, they may be silently skipped
3. **Pipeline Bypass**: Attackers might exploit misconfigured dependencies to skip mandatory checks
4. **Deployment Without Testing**: Broken dependency chains can lead to deployments without proper validation

### Understanding Job Dependencies

GitHub Actions uses the `needs:` keyword to define job dependencies. Jobs run in parallel by default, but `needs:` creates explicit execution order.

#### Basic Dependency Syntax

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm build

  test:
    needs: build  # Runs after 'build' completes
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: [build, test]  # Runs after both 'build' and 'test' complete
    runs-on: ubuntu-latest
    steps:
      - run: npm deploy
```

### What the Rule Detects

#### 1. Duplicate Job IDs in Needs

When the same job ID appears multiple times in a `needs:` array:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: make build

  test:
    needs: [build, lint, build]  # 'build' duplicated
    runs-on: ubuntu-latest
    steps:
      - run: make test
```

**Error Output:**

```bash
workflow.yml:10:5: job ID "build" duplicates in needs section [needs]
```

#### 2. References to Undefined Jobs

When `needs:` references a job that doesn't exist:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: make build

  deploy:
    needs: testing  # 'testing' job doesn't exist
    runs-on: ubuntu-latest
    steps:
      - run: make deploy
```

**Error Output:**

```bash
workflow.yml:9:5: job ID "deploy" needs job "testing" is not defined [needs]
```

#### 3. Duplicate Job Definitions

When the same job ID is defined multiple times:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: make build

  build:  # Duplicate job ID
    runs-on: ubuntu-latest
    steps:
      - run: npm build
```

**Error Output:**

```bash
workflow.yml:10:3: job ID "build" is already defined at line:4,col:3 [needs]
```

#### 4. Cyclic Dependencies

When jobs form a circular dependency chain:

```yaml
jobs:
  job-a:
    needs: job-c
    runs-on: ubuntu-latest
    steps:
      - run: echo "A"

  job-b:
    needs: job-a
    runs-on: ubuntu-latest
    steps:
      - run: echo "B"

  job-c:
    needs: job-b  # Creates cycle: a -> b -> c -> a
    runs-on: ubuntu-latest
    steps:
      - run: echo "C"
```

**Error Output:**

```bash
workflow.yml:4:3: cyclic dependency in needs section found: "job-a" -> "job-c", "job-b" -> "job-a", "job-c" -> "job-b" is detected cycle [needs]
```

### Safe Patterns

#### Pattern 1: Linear Dependency Chain

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm ci
      - run: npm run build

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: npm run deploy
```

#### Pattern 2: Fan-Out Pattern

Multiple jobs depending on a single job:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  unit-tests:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:unit

  integration-tests:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:integration

  e2e-tests:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:e2e
```

#### Pattern 3: Fan-In Pattern

Single job depending on multiple jobs:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - run: npm audit

  deploy:
    needs: [lint, test, security-scan]  # Waits for all three
    runs-on: ubuntu-latest
    steps:
      - run: npm run deploy
```

#### Pattern 4: Diamond Pattern

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  lint:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: [lint, test]  # Waits for both branches
    runs-on: ubuntu-latest
    steps:
      - run: npm run deploy
```

### Best Practices

#### 1. Keep Dependencies Explicit and Minimal

Only add dependencies that are truly necessary:

```yaml
# Good: Explicit, minimal dependencies
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    needs: build  # Only depends on build
    steps: [...]

# Avoid: Unnecessary transitive dependencies
jobs:
  test:
    needs: [build, lint, setup]  # If 'lint' already needs 'build', don't repeat
```

#### 2. Document Complex Dependencies

Add comments for non-obvious dependency relationships:

```yaml
jobs:
  deploy:
    # Requires both security scan and tests to pass before deployment
    needs: [security-scan, integration-tests]
    runs-on: ubuntu-latest
```

#### 3. Use Conditional Dependencies Carefully

```yaml
jobs:
  deploy:
    needs: test
    if: needs.test.result == 'success'  # Only deploy if tests passed
    runs-on: ubuntu-latest
```

### Relationship to Other Rules

- **[id]({{< ref "idRule.md" >}})**: Invalid job IDs will cause dependency resolution to fail
- **[workflow-call]({{< ref "workflowcall.md" >}})**: Reusable workflows have their own dependency considerations

### References

- [GitHub Docs: Workflow Syntax - needs](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idneeds)
- [GitHub Docs: Using jobs in a workflow](https://docs.github.com/en/actions/using-jobs/using-jobs-in-a-workflow)

### Configuration

This rule is enabled by default. To disable it:

```bash
sisakulint -ignore needs
```

Disabling this rule is **not recommended** as invalid job dependencies will cause workflow failures on GitHub.
