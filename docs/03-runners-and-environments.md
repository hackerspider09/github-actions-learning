# Chapter 3: Runners and Environments

A runner is the server environment that executes a specific job within a workflow. The runner environment is defined using the `runs-on` syntax at the job level.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
```

## 1. GitHub-Hosted Runners
GitHub provides managed virtual machines for workflow execution. These machines provide clean environments that are decommissioned after a job completes.

### Supported Operating Systems
- `ubuntu-latest`: Recommended for general Linux workloads.
- `windows-latest`: Required for Windows-specific dependencies (e.g., .NET Framework).
- `macos-latest`: Required for macOS or iOS application builds using Xcode.

### Pre-installed Software
GitHub-hosted runners include pre-installed software such as Docker, Node.js, Python, Git, and cloud provider CLIs, reducing the need for manual setup scripts.

## 2. Self-Hosted Runners
Self-hosted runners are physical or virtual machines managed independently of GitHub.

```yaml
jobs:
  build:
    runs-on: self-hosted
```

### Characteristics of Self-Hosted Runners
- Permit access to private networks, internal services, and proprietary hardware.
- Do not consume GitHub Actions compute minutes.
- Persist state across workflow runs, unlike ephemeral GitHub-hosted runners.
- Require manual maintenance, patching, and scaling.
- Present a security risk if used on public repositories where malicious code execution could impact internal infrastructure.

## 3. The Runner Workspace
Upon initialization, the runner workspace directory does not automatically contain the repository code. The repository contents must be explicitly cloned into the workspace using the `actions/checkout` action.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Verify Workspace
        run: ls -la
```
The absolute path of the workspace directory can be accessed via the `${{ github.workspace }}` context variable.
