---
title: "Environment Variable Rule"
weight: 1
---

### Environment Variable Rule Overview

This rule validates environment variable names in GitHub Actions workflows to ensure they follow proper naming conventions. It detects invalid characters in variable names that would cause workflow failures or unexpected behavior.

#### Key Features:

- **Invalid Character Detection**: Identifies environment variable names containing `&`, `=`, or spaces
- **Multi-Level Validation**: Checks environment variables at workflow, job, step, container, and service levels
- **Expression Handling**: Properly handles environment variable names that contain GitHub Actions expressions

### Security Impact

**Severity: Low (2/10)**

Invalid environment variable names primarily cause operational issues, but can also have security implications:

1. **Workflow Failures**: Variables with invalid characters will cause workflow parsing or runtime errors
2. **Injection Vectors**: Improperly named variables might indicate injection attempts
3. **Configuration Errors**: Typos in variable names can lead to misconfigured secrets or credentials
4. **Shell Interpretation Issues**: Special characters can cause unexpected shell behavior

### Understanding Environment Variables in GitHub Actions

Environment variables can be defined at multiple levels in GitHub Actions:

```yaml
# Workflow level
env:
  GLOBAL_VAR: "value"

jobs:
  build:
    # Job level
    env:
      JOB_VAR: "value"

    container:
      image: node:18
      # Container level
      env:
        CONTAINER_VAR: "value"

    services:
      postgres:
        image: postgres
        # Service level
        env:
          SERVICE_VAR: "value"

    steps:
      - name: Test
        # Step level
        env:
          STEP_VAR: "value"
        run: echo "$STEP_VAR"
```

### Example Vulnerable Workflow

Common environment variable naming issues:

```yaml
name: CI Build

on: [push]

env:
  MY VAR: "value"        # Contains space
  DB=CONNECTION: "value" # Contains equals sign
  API&KEY: "secret"      # Contains ampersand

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      BUILD FLAG: "true"  # Contains space

    steps:
      - name: Test
        env:
          TEST&MODE: "unit"  # Contains ampersand
        run: echo "Testing"
```

### What the Rule Detects

#### 1. Spaces in Variable Names

Environment variable names cannot contain spaces:

```yaml
env:
  MY VARIABLE: "value"  # Invalid: contains space
```

**Error Output:**

```bash
workflow.yml:5:3: Environment variable name '"MY VARIABLE"' is not formatted correctly. Please ensure that it does not include characters such as '&', '=', or spaces, as these are not allowed in variable names. [env-var]
```

#### 2. Equals Signs in Variable Names

Variable names cannot contain `=`:

```yaml
env:
  DB=HOST: "localhost"  # Invalid: contains equals sign
```

#### 3. Ampersands in Variable Names

Variable names cannot contain `&`:

```yaml
env:
  USER&PASS: "secret"  # Invalid: contains ampersand
```

### Safe Patterns

#### Pattern 1: Standard Naming Convention

Use uppercase with underscores:

```yaml
env:
  DATABASE_HOST: "localhost"
  DATABASE_PORT: "5432"
  API_KEY: ${{ secrets.API_KEY }}
  NODE_ENV: "production"
```

#### Pattern 2: Lowercase with Underscores

Some projects prefer lowercase:

```yaml
env:
  database_url: ${{ secrets.DATABASE_URL }}
  log_level: "debug"
  cache_enabled: "true"
```

#### Pattern 3: Dynamic Variable Names with Expressions

When using expressions, the rule allows dynamic naming:

```yaml
env:
  ${{ matrix.env_name }}: ${{ matrix.env_value }}
```

### Best Practices

#### 1. Use Standard Naming Conventions

Follow Unix/POSIX conventions for environment variable names:

```yaml
# Good: SCREAMING_SNAKE_CASE
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_ENDPOINT: "https://api.example.com"
  MAX_RETRY_COUNT: "3"

# Also acceptable: lowercase_snake_case
env:
  database_url: ${{ secrets.DATABASE_URL }}
```

#### 2. Avoid Special Characters

Only use alphanumeric characters and underscores:

```yaml
# Good
env:
  MY_VAR_123: "value"
  _INTERNAL_VAR: "value"

# Bad
env:
  MY-VAR: "value"    # Hyphens not recommended (works but unusual)
  MY.VAR: "value"    # Periods may cause issues
```

#### 3. Use Meaningful Names

Variable names should be descriptive:

```yaml
# Good: Descriptive names
env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  NPM_REGISTRY_URL: "https://registry.npmjs.org"
  BUILD_ARTIFACT_PATH: "./dist"

# Bad: Vague names
env:
  VAR1: "value"
  X: "something"
```

### References

- [GitHub Docs: Environment Variables](https://docs.github.com/en/actions/learn-github-actions/environment-variables)
- [GitHub Docs: Workflow Syntax - env](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#env)
- [POSIX Environment Variables](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap08.html)

### Configuration

This rule is enabled by default. To disable it:

```bash
sisakulint -ignore env-var
```

Disabling this rule is **not recommended** as invalid environment variable names will cause workflow failures or unexpected behavior.
