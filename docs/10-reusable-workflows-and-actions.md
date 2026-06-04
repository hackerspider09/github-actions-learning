# Chapter 10: Reusable Workflows and Composite Actions

Workflow definitions often contain repetitive logic across multiple repositories or jobs. GitHub Actions mitigates code duplication via Reusable Workflows and Composite Actions.

## 1. Reusable Workflows

A reusable workflow permits an entire workflow definition to be invoked by other workflows.

### Defining a Reusable Workflow
A workflow is made reusable by specifying the `workflow_call` trigger in the `on` block. Inputs and secrets required by the workflow are defined within this block.

```yaml
# File: .github/workflows/deploy-template.yaml
name: Deployment Template
on:
  workflow_call:
    inputs:
      target_env:
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying target environment: ${{ inputs.target_env }}"
```
*(Reference: `14.1-sub-workflow.yaml`)*

### Invoking a Reusable Workflow
A caller workflow executes the reusable workflow via the `uses` keyword within a job definition, supplying required inputs via `with`.

```yaml
# File: .github/workflows/caller-workflow.yaml
jobs:
  execute-deployment:
    uses: ./.github/workflows/deploy-template.yaml
    with:
      target_env: "production"
    secrets: inherit 
```
*(Reference: `14-workflow-input-output-and-reuable.yaml`)*

## 2. Composite Actions

Composite actions bundle a sequence of standard steps into a single reusable action. They are used to abstract repetitive setup or configuration sequences without delegating the entire job context.

### Defining a Composite Action
A composite action requires an `action.yml` file. The `runs` block must specify `using: "composite"`.

```yaml
# File: .github/actions/setup-environment/action.yml
name: 'Environment Setup'
description: 'Clones repository and installs dependencies'

runs:
  using: "composite"
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v3
      with:
        node-version: 18
    - run: npm ci
      shell: bash 
```
*Note: The `shell` attribute is mandatory for all `run` steps within a composite action.*

### Invoking a Composite Action
The composite action is invoked within a step sequence via the `uses` keyword pointing to the directory containing the `action.yml` file.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: ./.github/actions/setup-environment
      - run: npm test
```
