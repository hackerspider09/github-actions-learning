# Chapter 5: Variables, Secrets, and Environments

## Disambiguation: Two Meanings of "Environment"

The word "environment" appears in two completely different contexts within GitHub Actions. Confusing them is one of the most common sources of misunderstanding for anyone new to the platform.

**1. Environment Variables (`env:` block in YAML)**
Shell-level variables set inside a workflow file and injected into the runner's operating system environment. These are configuration values that scripts read during execution - no GitHub UI configuration required.

**2. GitHub Deployment Environments (configured in GitHub Settings)**
Named deployment targets (e.g., `development`, `staging`, `production`) that are created in the repository's GitHub Settings page. A deployment environment is a container for environment-specific secrets and variables, and can be protected with approval gates and branch restrictions. A job opts into an environment by declaring `environment: <name>`.

These two concepts are entirely separate. The rest of this chapter addresses each in turn.

---

## Part 1: Environment Variables

Environment variables are key-value pairs that a running shell script can read. In GitHub Actions, they are defined in the workflow YAML and are automatically injected into the runner's shell for the scope they are declared in.

### Variable Scoping
Variables resolve based on scope hierarchy. A variable defined at a lower scope overrides one with the same name defined at a higher scope.

- **Workflow Scope**: Defined at the top level of the YAML file. Available to all jobs and all steps.
- **Job Scope**: Defined inside a `jobs.<job-id>.env` block. Overrides workflow scope. Available to all steps in that job.
- **Step Scope**: Defined inside a specific step's `env` block. Overrides job scope. Available only to that step.

```yaml
env:
  SERVER_NAME: "Production"
  PORT: 80                   # Workflow-level default

jobs:
  test-job:
    env:
      PORT: 8080             # Overrides workflow-level PORT for this job
    runs-on: ubuntu-latest
    steps:
      - name: Default port
        run: echo "$SERVER_NAME on port $PORT"  # Output: Production on port 8080

      - name: Override for this step
        env:
          PORT: 3000         # Overrides job-level PORT for this step only
        run: echo "$SERVER_NAME on port $PORT"  # Output: Production on port 3000
```
*(Reference: `02-test-env.yaml`)*

### Accessing Variables: Two Methods

There are two distinct syntaxes for reading variables, and using the wrong one in the wrong place is a common mistake.

#### Method 1: Expression Syntax - `${{ env.VAR_NAME }}`
Evaluated by the GitHub Actions engine **before** the job is sent to the runner. Use this method when:
- Writing `if:` conditions
- Passing values to an action's `with:` inputs
- Constructing values in the workflow definition itself (e.g., `name:` fields)

```yaml
steps:
  - name: Conditional deploy
    if: ${{ env.SERVER_NAME == 'Production' }}
    uses: actions/checkout@v4
```

#### Method 2: Native Shell Variables - `$VAR_NAME`
Evaluated by the runner's shell at execution time. Use this method inside `run:` blocks. This is the **recommended approach** for shell scripts because injecting `${{ }}` expressions directly into a shell string can introduce script injection vulnerabilities - if the variable contains shell metacharacters or user-controlled content, the shell may interpret them.

```yaml
steps:
  - name: Safe shell usage
    run: echo "Target: $SERVER_NAME"
```

**Why `${{ env.VAR }}` inside `run:` is risky:**
```yaml
# Unsafe: if github.event.issue.title contains '$(rm -rf /)', the shell executes it
- run: echo "Issue title: ${{ github.event.issue.title }}"

# Safe: the value is passed as a variable, the shell never interpolates it as a command
- env:
    ISSUE_TITLE: ${{ github.event.issue.title }}
  run: echo "Issue title: $ISSUE_TITLE"
```

### Dynamic Variables (`$GITHUB_ENV`)
Each step runs in an isolated shell process. A variable set with `export` or inline assignment in one step does not exist in the next step. To persist a dynamically computed variable across steps, write it to the `$GITHUB_ENV` file.

```yaml
steps:
  - name: Calculate version
    run: |
      APP_VER="1.5.0-$(date +%Y%m%d)"
      echo "VERSION=$APP_VER" >> $GITHUB_ENV

  - name: Use version in next step
    run: echo "Deploying $VERSION"   # VERSION is now available
```
*(Reference: `03-test-github-env.yml`)*

---

## Part 2: Repository Secrets and Variables

Secrets and variables are values stored in GitHub's settings rather than in the workflow file. They are used when a value should not be committed to the repository (secrets) or when a configuration value needs to be shared across workflows without repeating it (variables).

### Secrets
Secrets are encrypted. Once created, the raw value cannot be retrieved through the GitHub UI - it can only be used or deleted. GitHub automatically redacts secret values from workflow logs so they are never printed in plain text.

Secrets are accessed via `${{ secrets.SECRET_NAME }}`.

```yaml
steps:
  - name: Authenticate with registry
    env:
      REGISTRY_TOKEN: ${{ secrets.REGISTRY_TOKEN }}
    run: docker login -u myuser --password "$REGISTRY_TOKEN" registry.example.com
```

Note the pattern: assign the secret to an environment variable in the `env:` block, then reference the environment variable in the shell command. This keeps the secret out of the shell's command string.

### Variables
Variables store non-sensitive configuration values. Unlike secrets, variable values are visible in logs and in the GitHub Settings UI. They are accessed via `${{ vars.VAR_NAME }}`.

```yaml
steps:
  - run: echo "Deploying app version ${{ vars.APP_VERSION }} to ${{ vars.TARGET_REGION }}"
```

### Where to Create Secrets and Variables
Both are created in the GitHub repository UI:

**Repository Settings → Secrets and variables → Actions**

From this page:
- **Secrets** tab: create encrypted secrets scoped to the repository.
- **Variables** tab: create plain-text configuration variables scoped to the repository.

These values are then available to all workflows in the repository via `${{ secrets.NAME }}` and `${{ vars.NAME }}`.

### Secret and Variable Scoping Precedence
The same secret or variable name can be defined at multiple levels. The most specific level takes precedence.

| Level | Scope | Access |
|---|---|---|
| Organization | All repositories in the organization | `${{ secrets.NAME }}` / `${{ vars.NAME }}` |
| Repository | All workflows in the repository | `${{ secrets.NAME }}` / `${{ vars.NAME }}` |
| Environment | Only jobs that declare that specific environment | `${{ secrets.NAME }}` / `${{ vars.NAME }}` |

An environment-level secret with the same name as a repository-level secret will override the repository value - but only for jobs that have declared `environment: <name>` in their definition.

---

## Part 3: GitHub Deployment Environments

A GitHub deployment environment is a **named configuration target** created in the repository's GitHub Settings. It is not a YAML concept - it is a GitHub platform feature.

**What an environment provides:**
- A set of secrets and variables that are specific to that deployment target and isolated from the rest of the repository.
- Protection rules that must be satisfied before any job targeting the environment is allowed to execute.
- A deployment record in the repository's Deployments tab, showing which commit was deployed to which environment and when.

**When to use environments:**
Use environments to model your deployment pipeline stages. A standard pattern is three environments - `development`, `staging`, `production` - each with their own database credentials, API keys, and infrastructure endpoints stored as environment secrets.

### Creating an Environment
Environments are created in **GitHub Settings → Environments → New environment**.

After creation, you can add environment-specific secrets and variables from the same page. These values are only accessible to jobs that declare that environment.

### Targeting an Environment from a Job
A job targets a deployment environment by declaring the `environment:` key.

```yaml
jobs:
  deploy-to-staging:
    runs-on: ubuntu-latest
    environment: staging          # This job can now access staging secrets/variables
    steps:
      - run: ./deploy.sh --target staging --db "${{ secrets.DB_PASSWORD }}"
        # secrets.DB_PASSWORD here resolves to the staging environment's secret,
        # not the repository-level secret of the same name (if one exists)
```

### Environment with a Deployment URL
A deployment URL can be specified alongside the environment name. GitHub displays this URL as a clickable link in the workflow run UI and in the repository's Deployments view, providing direct navigation to the deployed resource.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - run: ./deploy.sh
```
*(Reference: `11-secret-var-environmet.yaml`)*

### Environment Protection Rules
Protection rules are configured per environment in the GitHub Settings UI. A job targeting a protected environment must pass all active rules before its steps begin executing.

**Required Reviewers:**
One or more GitHub users or teams are designated as required approvers. When the workflow reaches a job targeting this environment, execution pauses. GitHub sends review notifications to the designated reviewers. The job only proceeds after an approver grants approval in the GitHub UI. If the request is rejected, the job is cancelled and the workflow fails.

This is the standard gate used to control production deployments - a human must explicitly approve before the production job runs.

**Wait Timer:**
A minimum wait duration (in minutes) is enforced before the job is permitted to start. This creates a deliberate delay - for example, a 10-minute wait on a production environment gives operators a window to cancel a deployment before it begins.

**Deployment Branches and Tags:**
Access to an environment can be restricted to specific branches or tag patterns. A workflow running on a branch that does not match the configured pattern (e.g., only `main` is allowed for `production`) will fail the environment check before the job executes.

**Combining Rules:**
All active protection rules are applied. A job must satisfy every rule before proceeding. For production, a typical configuration is: required reviewers + wait timer + branch restriction to `main`.

---

## The `GITHUB_TOKEN` Built-in Secret

Every workflow run is automatically provisioned with a `GITHUB_TOKEN` secret. GitHub generates it at the start of the run with permissions scoped to the repository. It expires when the run completes. No setup is required.

The token is accessed via `${{ secrets.GITHUB_TOKEN }}` and authenticates against the GitHub API. Common uses: creating releases, pushing commits, commenting on pull requests, managing deployments, and downloading artifacts from private repositories.

```yaml
steps:
  - name: Create GitHub Release
    uses: actions/create-release@v1
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    with:
      tag_name: ${{ github.ref }}
      release_name: Release ${{ github.ref }}
```

The default permissions of `GITHUB_TOKEN` are controlled by repository settings. For fine-grained control, use the `permissions:` block to restrict or expand what the token can do within a specific workflow or job. See [Chapter 15](15-permissions-and-github-token.md) for the complete `permissions:` reference.
