# Chapter 2: Events and Triggers

Workflows are triggered by events defined by the `on` syntax. 

## 1. Code Changes (Push & Pull Requests)
Workflows can be configured to run when code is pushed or when a pull request is modified.

### Push
Trigger a workflow upon a code push event:
```yaml
on:
  push:
```

**Filtering by Branch and Paths:**
Execution can be scoped to specific branches or paths:
```yaml
on:
  push:
    branches:
      - main
      - 'releases/**'
    branches-ignore:
      - 'experimental-**'
    paths:
      - 'src/**'
    paths-ignore:
      - 'docs/**'
```

### Pull Request
Trigger a workflow on pull request activities:
```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - main
```

## 2. Manual Triggers (`workflow_dispatch`)
The `workflow_dispatch` event allows workflows to be manually triggered from the GitHub UI or API. It supports input parameters.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod
      debug_mode:
        description: 'Enable debug logging'
        type: boolean
        default: false
```
Inputs are accessed within the workflow via `${{ inputs.<name> }}`. *(Reference: `06-test-inputs.yaml`)*.

## 3. Scheduled Triggers (Cron)
The `schedule` event triggers a workflow at scheduled times using POSIX cron syntax.

```yaml
on:
  schedule:
    - cron: "0 0 * * 0"
```
*(Reference: `13-cronjob.yaml`)*
*Note: Scheduled workflows run in UTC time and may experience delays during periods of high platform load.*

## 4. Workflow Run (`workflow_run`)
Workflows can be triggered by the completion of another workflow using the `workflow_run` event.

```yaml
on:
  workflow_run:
    workflows: ["Build Workflow"]
    types:
      - completed
```
To execute steps conditionally based on the upstream workflow's success or failure, evaluate the `conclusion` property:
```yaml
jobs:
  notify:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
```
*(Reference: `15-workflow-run-trigger.yaml`)*

## 5. Webhooks and Repository Events
Workflows can be triggered by various repository events:
- `issues`: Triggers on issue activity (e.g., opened, edited).
- `release`: Triggers when a release is published.
- `repository_dispatch`: Triggers via an external API payload.

```yaml
on:
  issues:
    types: [opened, edited]
```
