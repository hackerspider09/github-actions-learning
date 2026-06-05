# Chapter 14: Concurrency and Timeouts

## Concurrency Control

By default, GitHub Actions queues and runs every triggered workflow independently. On high-velocity repositories, this means multiple runs of the same workflow can execute simultaneously - for example, two pushes in rapid succession both trigger a deployment pipeline. Concurrency control prevents this by grouping runs and optionally cancelling stale ones.

### The `concurrency` Key

The `concurrency` block defines a group name. Only one workflow run sharing the same group name is permitted to execute at a time. A second run that enters the same group while the first is in-progress is either queued or, if `cancel-in-progress` is set, immediately cancelled.

```yaml
concurrency:
  group: production-deploy
  cancel-in-progress: true
```

**`cancel-in-progress: true`** - The in-progress run is cancelled and the new run proceeds immediately. This is the standard pattern for CI pipelines where only the most recent commit matters.

**`cancel-in-progress: false`** (default) - The new run waits in a queue until the current run completes. Suitable for deployment jobs where all queued runs must eventually complete.

### Dynamic Group Names
Hardcoding a group name causes all branches to share a single concurrency slot. A dynamic group name scoped to the branch or pull request is the standard pattern.

```yaml
# Scope concurrency to the triggering branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

With this configuration, two pushes to `feature/login` cancel each other, but a push to `feature/checkout` runs concurrently with `feature/login` because the group names differ.

```yaml
# Scope concurrency to the pull request number
concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

### Job-Level Concurrency
The `concurrency` key can be applied to the workflow level (as shown above) or to an individual job level. Job-level concurrency is useful when only a specific job (e.g., deployment) must be serialised, while other jobs (e.g., tests) can run in parallel.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    runs-on: ubuntu-latest
    needs: test
    concurrency:
      group: deploy-${{ github.ref }}
      cancel-in-progress: false   # Queue deployments; never cancel mid-deploy
    steps:
      - run: ./deploy.sh
```

## Timeouts

### Job Timeout (`timeout-minutes`)
By default, a GitHub-hosted job is allowed to run for up to **6 hours** before GitHub forcefully terminates it. For most workflows this is excessive. Setting `timeout-minutes` at the job level defines a hard limit. If the job has not completed within the specified duration, it is cancelled and marked as failed.

```yaml
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - run: ./run-integration-tests.sh
```

### Step Timeout (`timeout-minutes`)
Individual steps can also be given a timeout. A step-level timeout applies only to that step; the job continues if the step times out and `continue-on-error` is set.

```yaml
steps:
  - name: Health Check
    run: ./wait-for-service.sh
    timeout-minutes: 5
    continue-on-error: true

  - name: Run Tests
    run: npm test
    timeout-minutes: 20
```

### Timeout Precedence
If both a job timeout and a step timeout are defined, whichever limit is reached first takes effect. A step timeout does not extend a job timeout.

| Scope | Default | Maximum |
|---|---|---|
| Workflow | None | Determined by organization/repository billing settings |
| Job | 360 minutes (6 hours) | 360 minutes |
| Step | No default | Bound by job timeout |
