# Chapter 5: Variables, Secrets, and Contexts

## Environment Variables (`env`)
Environment variables can be defined natively for use in shell scripts via the `env` block.

### Variable Scoping
Variables resolve based on scope hierarchy. Lower scopes override higher scopes if variables share the same name.
- **Workflow Scope**: Defined at the top level; available to all jobs and steps.
- **Job Scope**: Overrides workflow scope; available to all steps in the job.
- **Step Scope**: Overrides job scope; available only to the specific step.

```yaml
env:
  SERVER_NAME: "Production"
  PORT: 80

jobs:
  test-job:
    env:
      PORT: 8080 
    runs-on: ubuntu-latest
    steps:
      - env:
          PORT: 3000
        run: echo "Connecting to $SERVER_NAME on port $PORT"
```
*(Reference: `02-test-env.yaml`)*

## Methods of Accessing Variables
There are multiple methods to access variables depending on the context of execution. Utilizing the correct method is critical for workflow security and functionality.

### 1. Expression Syntax (`${{ env.VAR_NAME }}`)
Expression syntax is interpolated by the GitHub Actions engine *before* the script is sent to the runner. This method is required when evaluating `if` conditionals or when passing variables to actions via `with` parameters.
```yaml
steps:
  - name: Conditional Execution
    if: ${{ env.SERVER_NAME == 'Production' }}
    uses: actions/checkout@v4
```

### 2. Native Shell Variables (`$VAR_NAME`)
When executing commands inside a `run` block, variables should be accessed using the runner's native shell syntax (e.g., `$VAR_NAME` or `${VAR_NAME}` in Bash). The `env` block natively exports variables to the runner's OS environment. Relying on native shell variables mitigates script injection vulnerabilities that can occur when unescaped expression syntax (`${{ }}`) is injected directly into a shell execution context.
```yaml
steps:
  - name: Native Shell Execution
    run: echo "Target Server: $SERVER_NAME"
```

## Dynamic Variables (`$GITHUB_ENV`)
Environment variables declared in a `run` step do not persist to subsequent steps because each step runs in an isolated shell. To persist a dynamically generated variable, write it to the `$GITHUB_ENV` environment file.

```yaml
steps:
  - name: Calculate Version
    run: |
      APP_VER="1.5.0"
      echo "VERSION=$APP_VER" >> $GITHUB_ENV

  - name: Use the Variable
    run: |
      echo "Deploying version $VERSION"
```
*(Reference: `03-test-github-env.yml`)*

## Repository Variables and Secrets
Sensitive data and configuration strings should be managed via GitHub Settings rather than hardcoded in the repository.

### Secrets
Secrets are encrypted environment variables used for sensitive data (e.g., API keys, passwords). They are accessed via `${{ secrets.SECRET_NAME }}`. GitHub automatically masks secret values in logs.

```yaml
steps:
  - name: Authenticate API
    run: ./login.sh --token ${{ secrets.API_TOKEN }}
```

### Configuration Variables
Non-sensitive configuration data (e.g., target endpoints) can be stored as repository variables and accessed via `${{ vars.VAR_NAME }}`.

## Environments and Protection Rules
Environments can be configured in GitHub Settings to isolate secrets and variables specific to deployment targets (e.g., `development`, `staging`, `production`).

A job targeting an environment will inherit the secrets and variables specific to that environment.

```yaml
jobs:
  deploy-to-prod:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh --password ${{ secrets.DB_PASSWORD }}
```
*(Reference: `11-secret-var-environmet.yaml`)*

### Protection Rules
Environments support protection rules, such as required manual approvals. If an environment is configured with required reviewers, workflows targeting that environment will pause execution until approval is granted via the GitHub UI.
