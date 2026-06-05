# Chapter 10: Reusable Workflows and Custom Actions

As a CI/CD system grows, the same logic appears in multiple places. The same build steps get copied into every repository. The same deployment sequence gets duplicated across staging and production workflows. GitHub Actions provides three distinct mechanisms to eliminate this duplication: Reusable Workflows, Composite Actions, and purpose-built custom actions.

Understanding which mechanism to use requires understanding what each one is designed for.

| Mechanism | What it encapsulates | What the caller gets |
|---|---|---|
| Reusable Workflow | An entire workflow with one or more jobs | A full job block - the caller cannot add steps around it |
| Composite Action | A sequence of steps | A single `uses:` step the caller can combine with other steps |
| Custom Action (Docker/JS) | Arbitrary logic in any language | A single `uses:` step; full isolation from the runner environment |

## Part 1: Reusable Workflows

### What They Are For

A reusable workflow is a complete workflow file that other workflows can invoke as if it were a job. The canonical use case is a deployment pipeline: you define the deploy sequence once in a shared workflow, and every service-specific workflow calls it with different parameters.

### Defining a Reusable Workflow

To make a workflow reusable, add `workflow_call` to its `on:` block. This declares that the workflow can be triggered by another workflow (in addition to any other triggers it may have).

```yaml
# File: .github/workflows/deploy.yml
name: Deploy

on:
  workflow_call:
    inputs:
      environment:
        description: 'Target environment name'
        required: true
        type: string
      image_tag:
        description: 'Docker image tag to deploy'
        required: true
        type: string
      dry_run:
        description: 'Run without making changes'
        required: false
        type: boolean
        default: false
    secrets:
      DEPLOY_KEY:
        description: 'SSH key for deployment server'
        required: true
    outputs:
      deployment_url:
        description: 'URL of the deployed application'
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.get-url.outputs.url }}
    steps:
      - name: Deploy application
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
        run: |
          if [[ "${{ inputs.dry_run }}" == "true" ]]; then
            echo "Dry run: would deploy ${{ inputs.image_tag }} to ${{ inputs.environment }}"
          else
            ./deploy.sh --env ${{ inputs.environment }} --tag ${{ inputs.image_tag }}
          fi

      - name: Get deployment URL
        id: get-url
        run: echo "url=https://${{ inputs.environment }}.app.example.com" >> $GITHUB_OUTPUT
```
*(Reference: `14.1-sub-workflow.yaml`)*

**Input types:** `string`, `boolean`, `number`. All inputs are accessed via `${{ inputs.name }}`.

**Secrets in reusable workflows:** Secrets cannot be passed via `inputs`. They are declared separately under `secrets:` in the `workflow_call` block and accessed via `${{ secrets.NAME }}`.

**Outputs from reusable workflows:** A reusable workflow can pass data back to the caller. Outputs are declared under `outputs:` in `workflow_call` and must map to a job output within the workflow.

### Invoking a Reusable Workflow

A caller workflow uses the reusable workflow by referencing it in a job's `uses:` field instead of `runs-on:`.

```yaml
# File: .github/workflows/release.yml
name: Release Pipeline

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.build.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - id: build
        run: |
          TAG="${{ github.ref_name }}"
          docker build -t myapp:$TAG .
          docker push myapp:$TAG
          echo "tag=$TAG" >> $GITHUB_OUTPUT

  deploy-staging:
    needs: build
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
      image_tag: ${{ needs.build.outputs.image_tag }}
      dry_run: false
    secrets:
      DEPLOY_KEY: ${{ secrets.STAGING_DEPLOY_KEY }}

  deploy-production:
    needs: [build, deploy-staging]
    uses: ./.github/workflows/deploy.yml
    with:
      environment: production
      image_tag: ${{ needs.build.outputs.image_tag }}
    secrets:
      DEPLOY_KEY: ${{ secrets.PROD_DEPLOY_KEY }}
```
*(Reference: `14-workflow-input-output-and-reuable.yaml`)*

**Reading outputs from a reusable workflow:**
```yaml
jobs:
  after-deploy:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deployed to ${{ needs.deploy-staging.outputs.deployment_url }}"
```

### `secrets: inherit`

When a caller workflow passes `secrets: inherit`, all secrets available to the caller are automatically forwarded to the reusable workflow. The reusable workflow can access them by name without the caller explicitly listing them.

```yaml
uses: ./.github/workflows/deploy.yml
with:
  environment: production
secrets: inherit    # All caller secrets are available to the called workflow
```

Use this pattern when the caller and the reusable workflow are in the same repository and trust boundary. When calling a reusable workflow in a different repository, explicitly list only the secrets that are needed.

### Referencing Reusable Workflows from Other Repositories

```yaml
uses: my-org/shared-workflows/.github/workflows/deploy.yml@main
with:
  environment: production
secrets:
  DEPLOY_KEY: ${{ secrets.PROD_DEPLOY_KEY }}
```

The format is `{owner}/{repo}/.github/workflows/{filename}@{ref}`. The ref can be a branch, tag, or commit SHA.

## Part 2: Composite Actions

### What They Are For

Composite actions encapsulate a sequence of steps. Unlike reusable workflows, a composite action is invoked as a single step inside a job, and the caller can add other steps before and after it. This makes them suitable for setup sequences: checking out code, installing a language runtime, and installing dependencies can all be wrapped into one `uses:` reference.

### Composite Action File Structure

A composite action requires a file named `action.yml` placed in a directory. For actions used within the same repository, the convention is to place them in `.github/actions/<action-name>/`.

```
.github/
  actions/
    setup-project/
      action.yml      <-- the composite action definition
```

### Defining a Composite Action with Inputs and Outputs

```yaml
# File: .github/actions/setup-project/action.yml
name: 'Setup Project'
description: 'Checks out the repository, installs Node, and installs dependencies'

inputs:
  node-version:
    description: 'Node.js version to install'
    required: false
    default: '20'
  install-command:
    description: 'Command to install dependencies'
    required: false
    default: 'npm ci'

outputs:
  cache-hit:
    description: 'Whether the dependency cache was hit'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: 'composite'
  steps:
    - name: Check out repository
      uses: actions/checkout@v4

    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Cache dependencies
      id: cache
      uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

    - name: Install dependencies
      if: steps.cache.outputs.cache-hit != 'true'
      run: ${{ inputs.install-command }}
      shell: bash    # Required for all 'run' steps in composite actions
```

**Important:** Every `run` step inside a composite action must declare a `shell:` property. This is required because composite actions can run on any OS and GitHub does not assume a default shell for them.

### Invoking a Composite Action

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Set up environment
        id: setup
        uses: ./.github/actions/setup-project
        with:
          node-version: '20'

      - name: Check cache result
        run: echo "Cache hit: ${{ steps.setup.outputs.cache-hit }}"

      - name: Run tests
        run: npm test

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: ./.github/actions/setup-project  # Reuse setup in another job
      - run: npm run build
```

The `.` in `./.github/actions/setup-project` means the action is in the current repository. The path points to the directory containing `action.yml`.

### Composite Actions vs Reusable Workflows: Choosing Between Them

Use a composite action when:
- You want to reuse a sequence of steps inside different jobs, alongside other steps.
- The setup logic is the same across jobs but the jobs do different things after setup.
- You need to accept inputs and return outputs at the step level.

Use a reusable workflow when:
- The entire job sequence is what you want to reuse.
- Different teams or repositories need to run the same deployment or build pipeline.
- You want environment-level secrets and protection rules to apply to the reused logic.
