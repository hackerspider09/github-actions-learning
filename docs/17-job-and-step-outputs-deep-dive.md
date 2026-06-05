# Chapter 17: Job and Step Outputs - Complete Reference

Data produced during a workflow run can be made available to downstream steps or jobs through a structured output mechanism. This chapter consolidates all aspects of the output system: how data is written, how it is read, the fields that describe step execution results, and how job results are consumed by dependent jobs.

## The Output File Architecture

GitHub Actions uses purpose-specific environment files on the runner filesystem to communicate state between steps. Steps write to these files; the Actions runner process reads them after each step completes and updates the appropriate context objects.

| File Variable | Written by step to | Effect |
|---|---|---|
| `$GITHUB_OUTPUT` | Append `KEY=VALUE` | Registers a named output on the step; readable by subsequent steps and by the job's `outputs:` block |
| `$GITHUB_ENV` | Append `KEY=VALUE` | Persists an environment variable for all subsequent steps in the job |
| `$GITHUB_PATH` | Append a directory path | Adds a directory to `$PATH` for all subsequent steps |
| `$GITHUB_STEP_SUMMARY` | Append Markdown content | Renders custom content on the workflow run summary page |

## Step Outputs (`$GITHUB_OUTPUT`)

### Writing a Step Output
A step must have an `id` defined. The output is written by appending `KEY=VALUE` to the `$GITHUB_OUTPUT` file.

```yaml
steps:
  - name: Generate Build Tag
    id: build-meta
    run: |
      TAG="${{ github.ref_name }}-${{ github.run_number }}"
      echo "build_tag=$TAG" >> $GITHUB_OUTPUT
```

### Reading a Step Output
A subsequent step within the same job reads the output via `${{ steps.<step-id>.outputs.<key> }}`.

```yaml
  - name: Use Build Tag
    run: docker build -t "my-app:${{ steps.build-meta.outputs.build_tag }}" .
```
*(Reference: `04-test-step-output.yml`)*

### Multiline Step Outputs
Values containing newlines cannot use the `KEY=VALUE` format. A heredoc-style delimiter is used instead.

```yaml
steps:
  - name: Generate Changelog
    id: changelog
    run: |
      EOF=$(openssl rand -hex 8)
      echo "content<<$EOF" >> $GITHUB_OUTPUT
      echo "- Fixed login bug" >> $GITHUB_OUTPUT
      echo "- Added dark mode" >> $GITHUB_OUTPUT
      echo "$EOF" >> $GITHUB_OUTPUT

  - name: Use Changelog
    run: |
      echo "${{ steps.changelog.outputs.content }}"
```

The delimiter (`$EOF` in this example) must be a string that does not appear in the value itself. Using a random string prevents accidental termination of the block.

## Step Execution Status Fields

Every step that executes records two status fields in the `steps` context. These fields are accessible in subsequent steps via `${{ steps.<step-id>.outcome }}` and `${{ steps.<step-id>.conclusion }}`.

### `outcome`
The actual result of the step as returned by the command or action. This reflects what happened - regardless of any `continue-on-error` configuration.

### `conclusion`
The effective result after `continue-on-error` processing. GitHub uses `conclusion` to determine whether the job has failed.

**Possible values for both fields:** `success`, `failure`, `cancelled`, `skipped`

### Effect of `continue-on-error` on Both Fields

| Scenario | `outcome` | `conclusion` |
|---|---|---|
| Step succeeds | `success` | `success` |
| Step fails, no `continue-on-error` | `failure` | `failure` |
| Step fails, `continue-on-error: true` | `failure` | `success` |
| Step is skipped by `if:` condition | `skipped` | `skipped` |

When a step fails with `continue-on-error: true`:
- `outcome` records `failure` - preserving the true result.
- `conclusion` records `success` - because the error was explicitly suppressed.
- The job status remains `success`, so `failure()` in subsequent steps evaluates to `false`.

To respond to the true failure of a step with `continue-on-error`, check `steps.<id>.outcome` rather than using `failure()`.

```yaml
steps:
  - name: Optional Scan
    id: scan
    run: ./security-scan.sh
    continue-on-error: true

  # failure() returns false here because the job is still success
  # Use outcome to detect the actual failure
  - name: Report Scan Result
    run: |
      if [[ "${{ steps.scan.outcome }}" == "failure" ]]; then
        echo "Scan found issues - review required"
      else
        echo "Scan passed"
      fi
```
*(Reference: `08-test-conditions.yaml`, `17-failure-workflow.yaml`)*

## Job Outputs

### Defining Job Outputs
Step outputs are not automatically available to other jobs. To expose a step output beyond the current job, it must be mapped to a job-level output in the `outputs:` block.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.build-meta.outputs.build_tag }}
      build_status: ${{ steps.build-meta.outputs.status }}
    steps:
      - name: Build
        id: build-meta
        run: |
          echo "build_tag=v1.0.0-${{ github.run_number }}" >> $GITHUB_OUTPUT
          echo "status=success" >> $GITHUB_OUTPUT
```

### Consuming Job Outputs
A downstream job accesses the output via `needs.<job-id>.outputs.<key>`. The consuming job must declare `needs` to ensure execution order and to gain access to the outputs context.

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - run: |
          echo "Deploying image: ${{ needs.build.outputs.image_tag }}"
          echo "Build status: ${{ needs.build.outputs.build_status }}"
```
*(Reference: `05-test-job-output.yaml`)*

## Job Result Values

When a job completes, GitHub assigns it a `result` value. Downstream jobs access this via `needs.<job-id>.result`.

| Value | Meaning |
|---|---|
| `success` | All steps completed without error. |
| `failure` | At least one step failed without `continue-on-error`. |
| `cancelled` | The run was cancelled before the job completed. |
| `skipped` | The job's own `if:` condition evaluated to `false`. |

### Using Job Result in Conditional Logic

**Important default behaviour:** When a job declares `needs`, its implicit `if:` condition is `success()`. This means the job is skipped if any upstream job fails. To override this and run the job regardless of upstream failure, an explicit `if:` condition must be provided.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh

  # This job will NOT run if deploy fails, because the default is success()
  next-step:
    runs-on: ubuntu-latest
    needs: deploy
    steps:
      - run: ./post-deploy.sh

  # This job WILL run when deploy fails, because we override the default
  rollback:
    runs-on: ubuntu-latest
    needs: deploy
    if: needs.deploy.result == 'failure'
    steps:
      - run: ./rollback.sh

  # This job runs regardless of the deploy outcome
  report:
    runs-on: ubuntu-latest
    needs: deploy
    if: always()
    steps:
      - run: |
          echo "Deploy result: ${{ needs.deploy.result }}"
```

### Evaluating Multiple Upstream Jobs

```yaml
jobs:
  report:
    runs-on: ubuntu-latest
    needs: [build, test, security]
    if: always()
    steps:
      - run: |
          echo "Build:    ${{ needs.build.result }}"
          echo "Test:     ${{ needs.test.result }}"
          echo "Security: ${{ needs.security.result }}"
```
*(Reference: `05-test-job-output.yaml`, `09-test-needs-if.yaml`)*

## Output Data Flow Summary

```
Step run: block
  └── writes to $GITHUB_OUTPUT
        └── becomes: steps.<id>.outputs.<key>
              └── mapped in job outputs: block
                    └── becomes: needs.<job-id>.outputs.<key>
                          └── consumed by downstream jobs via needs context
```
