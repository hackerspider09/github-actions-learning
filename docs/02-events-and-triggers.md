# Chapter 2: Events and Triggers

A workflow does nothing on its own. It waits for an event. The `on:` key defines which events a workflow responds to. When one of those events occurs in the repository, GitHub reads the workflow file and schedules a run.

A workflow can listen to multiple events simultaneously. Any one of them is sufficient to trigger a run.

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
```

This workflow runs when: code is pushed to `main`, a pull request targets `main`, or a user manually triggers it from the GitHub UI.

## 1. Push

The `push` event fires whenever commits are pushed to the repository, including pushes that create a new branch.

```yaml
on:
  push:
```

With no filters, this runs on every push to every branch. That includes feature branches, hotfix branches, and every intermediate commit. For most workflows, unfiltered push triggers too broadly.

### Filtering by Branch

Use `branches` to restrict execution to specific branches. Pattern matching with `**` (any number of characters across slashes) and `*` (any characters except `/`) is supported.

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'       # Matches release/1.0, release/v1/1.0, release/2.0, etc.
      - 'hotfix/*'         # Matches hotfix/login but NOT hotfix/auth/token
```

Use `branches-ignore` to run on all branches except specified ones. You cannot use `branches` and `branches-ignore` together in the same event.

```yaml
on:
  push:
    branches-ignore:
      - 'dependabot/**'    # Skip automated dependency update branches
```

### Filtering by Path

Use `paths` to run only when specific files change. This prevents triggering a build pipeline when someone updates only documentation.

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
```

Use `paths-ignore` to skip the workflow when only certain files change.

```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'
      - '**.md'
      - '.github/CODEOWNERS'
```

When both `branches` and `paths` are specified, both conditions must be true for the workflow to run. A push to `main` that only changes `docs/` would not trigger the workflow above.

### Filtering by Tag

```yaml
on:
  push:
    tags:
      - 'v*'              # Triggers on v1.0.0, v2.3.1, v10.0.0-beta, etc.
    tags-ignore:
      - 'v*-beta'         # All tags except beta releases
```

## 2. Pull Request

The `pull_request` event fires when activity occurs on a pull request. By default (with no `types` filter), it fires when a PR is opened, synchronized (new commits pushed), or reopened.

```yaml
on:
  pull_request:
    branches: [main]
```

### Activity Types

The `types` filter controls which PR activities trigger the workflow. The available types are:

| Type | When it fires |
|---|---|
| `opened` | A new pull request is created |
| `synchronize` | New commits are pushed to an existing pull request |
| `reopened` | A closed pull request is reopened |
| `closed` | A pull request is closed (merged or dismissed) |
| `ready_for_review` | A draft PR is marked as ready for review |
| `converted_to_draft` | A PR is converted back to draft |
| `labeled` | A label is added to the PR |
| `unlabeled` | A label is removed from the PR |
| `assigned` | A user is assigned to the PR |
| `review_requested` | A review is requested |

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]  # The default behavior
```

```yaml
# Only run when a PR is merged (closed + merged)
on:
  pull_request:
    types: [closed]

jobs:
  post-merge:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - run: echo "PR was merged, not just closed"
```

### Pull Requests from Forks

When an external contributor forks a public repository and opens a pull request, GitHub treats this differently for security reasons. A workflow triggered by `pull_request` from a fork runs with:

- **Read-only access** to the repository (the `GITHUB_TOKEN` cannot write)
- **No access to secrets** (secrets are not passed to the workflow)

This is intentional. Without this protection, any external contributor could open a PR and exfiltrate repository secrets through the workflow.

If your CI workflow needs to post a comment back to the pull request (which requires write access), the common pattern is to run the tests using `pull_request` (no secrets, safe), save results as an artifact, then have a second workflow triggered by `workflow_run` that downloads the artifact and posts the comment (with write access, runs from the base branch).

## 3. Manual Triggers (`workflow_dispatch`)

`workflow_dispatch` allows a workflow to be started manually from the GitHub UI, the GitHub CLI, or the GitHub API. It optionally accepts typed input parameters.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target deployment environment'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
      image_tag:
        description: 'Docker image tag to deploy'
        required: true
        type: string
      dry_run:
        description: 'Simulate deployment without making changes'
        type: boolean
        default: false
      version_override:
        description: 'Override the version number (leave empty to use default)'
        required: false
        type: string
        default: ''
```

Input types: `string`, `boolean`, `choice`, `number`, `environment`.

Inputs are accessed in the workflow via `${{ inputs.name }}`.

```yaml
steps:
  - run: |
      echo "Deploying to: ${{ inputs.environment }}"
      echo "Image tag:    ${{ inputs.image_tag }}"
      echo "Dry run:      ${{ inputs.dry_run }}"
```
*(Reference: `06-test-inputs.yaml`)*

### Triggering `workflow_dispatch` from the GitHub CLI

```bash
gh workflow run deploy.yml \
  --field environment=staging \
  --field image_tag=v1.2.3 \
  --field dry_run=false
```

### Triggering via the GitHub API

```bash
curl -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/OWNER/REPO/actions/workflows/deploy.yml/dispatches" \
  -d '{"ref":"main","inputs":{"environment":"staging","image_tag":"v1.2.3"}}'
```

## 4. Scheduled Triggers (`schedule`)

The `schedule` event triggers a workflow at specified times using standard cron syntax. Scheduled workflows always run on the default branch of the repository.

```yaml
on:
  schedule:
    - cron: '0 2 * * 1-5'   # 02:00 UTC, Monday through Friday
    - cron: '0 0 * * 0'     # Midnight UTC every Sunday
```

Multiple cron expressions can be defined. Each one is an independent trigger.

### Cron Syntax

```
 *  *  *  *  *
 |  |  |  |  |
 |  |  |  |  +-- Day of week (0-7, 0 and 7 are Sunday)
 |  |  |  +----- Month (1-12)
 |  |  +-------- Day of month (1-31)
 |  +----------- Hour (0-23)
 +-------------- Minute (0-59)
```

| Expression | Meaning |
|---|---|
| `0 0 * * *` | Every day at midnight UTC |
| `0 */6 * * *` | Every 6 hours |
| `0 9 * * 1-5` | 09:00 UTC on weekdays |
| `0 0 1 * *` | First day of every month at midnight |
| `*/15 * * * *` | Every 15 minutes |

**Important:** Scheduled workflows run in UTC. They may be delayed by up to several minutes during periods of high platform load. The minimum interval is 5 minutes; GitHub may not honor schedules shorter than that.

**Scheduled workflows on inactive repositories:** GitHub disables scheduled workflows on repositories that have had no activity (commits, PRs, issues) for 60 days to reduce unnecessary compute usage.
*(Reference: `13-cronjob.yaml`)*

## 5. Workflow Run (`workflow_run`)

`workflow_run` triggers a workflow when another named workflow completes, starts, or is requested. It is used to create multi-stage pipelines where workflows chain together.

```yaml
on:
  workflow_run:
    workflows: ['CI Pipeline']   # Must match the name: field of the upstream workflow
    types: [completed]           # 'completed', 'requested', or 'in_progress'
    branches: [main]             # Optional: only trigger for runs on these branches
```

The triggering workflow runs from the **default branch**, not from the branch of the upstream workflow. This means it always has access to repository secrets, making it suitable for tasks that need write access after a fork PR's CI completes.

### Filtering by Upstream Conclusion

Use `github.event.workflow_run.conclusion` to conditionally execute steps based on whether the upstream workflow succeeded or failed.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - run: ./deploy.sh

  notify-failure:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    steps:
      - run: ./send-alert.sh
```

Possible conclusion values: `success`, `failure`, `cancelled`, `timed_out`, `action_required`, `neutral`, `skipped`, `stale`.
*(Reference: `15-workflow-run-trigger.yaml`)*

## 6. Repository Dispatch (`repository_dispatch`)

`repository_dispatch` allows an external system to trigger a workflow via an HTTP request to the GitHub API. It is used to integrate GitHub Actions with external tools, custom dashboards, or other CI systems.

```yaml
on:
  repository_dispatch:
    types: [deploy-triggered, data-pipeline-complete]
```

The `types` filter matches the `event_type` field in the API request. If omitted, all `repository_dispatch` events trigger the workflow.

### Sending a `repository_dispatch` Event

```bash
curl -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/OWNER/REPO/dispatches" \
  -d '{
    "event_type": "deploy-triggered",
    "client_payload": {
      "image_tag": "v1.5.0",
      "environment": "production"
    }
  }'
```

The `client_payload` object is available in the workflow via `github.event.client_payload`.

```yaml
steps:
  - run: |
      echo "Image tag:   ${{ github.event.client_payload.image_tag }}"
      echo "Environment: ${{ github.event.client_payload.environment }}"
```

## 7. Other Common Events

| Event | When it fires | Common use |
|---|---|---|
| `issues` | Issue opened, closed, labeled, commented, etc. | Auto-label, auto-assign, triage workflows |
| `issue_comment` | Comment posted on an issue or pull request | Chatops: trigger actions from PR comments |
| `release` | Release created, published, edited, deleted | Build release artifacts, publish to registries |
| `create` | Branch or tag created | Notify on new release branch creation |
| `delete` | Branch or tag deleted | Clean up associated resources |
| `fork` | Repository is forked | Notify repository owner |
| `watch` | Repository is starred | Engagement tracking |
| `deployment` | A deployment is created via API | Chain with external deployment systems |
| `page_build` | GitHub Pages build completes | Post-pages-deploy notifications |

```yaml
# Run on issue or PR comments that contain /deploy
on:
  issue_comment:
    types: [created]

jobs:
  chatops:
    if: contains(github.event.comment.body, '/deploy')
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy triggered by comment from ${{ github.event.comment.user.login }}"
```

## 8. Multiple Events and Combining Triggers

Multiple events and their filters are defined as siblings under `on:`.

```yaml
on:
  push:
    branches: [main]
    paths: ['src/**']
  pull_request:
    branches: [main]
    types: [opened, synchronize]
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * 1'
```

Each event is evaluated independently. If a push to `main` modifies `src/` files, the `push` condition is satisfied and the workflow runs. A manual click of the Run Workflow button satisfies the `workflow_dispatch` condition independently.

### Checking Which Event Triggered the Run

Use `github.event_name` to branch logic inside a workflow based on which event fired.

```yaml
steps:
  - name: Identify trigger
    run: echo "Triggered by: ${{ github.event_name }}"

  - name: Deploy only on push to main
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    run: ./deploy.sh

  - name: Run full test suite on schedule
    if: github.event_name == 'schedule'
    run: npm run test:full
```
