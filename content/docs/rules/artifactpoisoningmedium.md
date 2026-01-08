---
title: "Artifact Poisoning Rule (Medium)"
weight: 9
---

### Artifact Poisoning Rule (Medium) Overview

This rule detects potential artifact poisoning vulnerabilities when third-party artifact download actions are used in workflows triggered by untrusted events. Unlike the critical severity rule which focuses on unsafe extraction paths with the official `actions/download-artifact`, this medium severity rule targets third-party actions that may have unsafe default behaviors.

#### Key Features:

- **Third-Party Action Detection**: Identifies uses of non-official artifact download actions (e.g., `dawidd6/action-download-artifact`)
- **Untrusted Trigger Context**: Checks for workflows triggered by `workflow_run`, `pull_request_target`, or `issue_comment`
- **Heuristic Detection**: Uses naming patterns to catch new or unknown third-party artifact actions
- **Auto-fix Support**: Automatically adds safe extraction paths using `${{ runner.temp }}/artifacts`

### Security Impact

**Severity: Medium (6/10)**

Third-party artifact download actions in untrusted contexts represent a medium security risk:

1. **Default Behavior Risk**: Third-party actions may extract artifacts directly to the workspace by default
2. **Untrusted Source**: Artifacts from `workflow_run` or `pull_request_target` may originate from untrusted PRs
3. **File Overwriting**: Without explicit safe paths, malicious artifacts can overwrite existing files
4. **Supply Chain Vector**: Compromised workflows can inject malicious content into privileged contexts

### Example Vulnerable Workflow

```yaml
name: Process PR Results

on:
  workflow_run:
    workflows: ["PR Build"]
    types: [completed]

jobs:
  process:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # VULNERABLE: Third-party action with workflow_run trigger
      - name: Download PR artifacts
        uses: dawidd6/action-download-artifact@v2
        with:
          name: pr_number
          # No path specified - may extract to workspace root!

      # DANGEROUS: Executes script that may be overwritten
      - name: Process results
        run: |
          sh ./process.sh
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Example Output

```bash
$ sisakulint

.github/workflows/process.yaml:12:9: artifact poisoning risk: third-party action "dawidd6/action-download-artifact@v2" downloads artifacts in workflow with untrusted triggers (workflow_run) without safe extraction path. [artifact-poisoning-medium]
     12 👈|      - name: Download PR artifacts
```

### Auto-fix Support

```bash
sisakulint -fix dry-run
sisakulint -fix on
```

After auto-fix:

```yaml
- name: Download PR artifacts
  uses: dawidd6/action-download-artifact@v2
  with:
    name: pr_number
    path: ${{ runner.temp }}/artifacts
```

### Best Practices

1. **Always Specify Safe Extraction Paths**
2. **Validate Artifact Content Before Use**
3. **Prefer Official Actions When Possible**
4. **Separate Untrusted and Privileged Workflows**

### See Also

- [CodeQL: Artifact Poisoning (Medium)](https://codeql.github.com/codeql-query-help/actions/actions-artifact-poisoning-medium/)
- [GitHub Security Lab: Preventing Pwn Requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)

{{< popup_link2 href="https://codeql.github.com/codeql-query-help/actions/actions-artifact-poisoning-medium/" >}}

{{< popup_link2 href="https://securitylab.github.com/research/github-actions-preventing-pwn-requests/" >}}
