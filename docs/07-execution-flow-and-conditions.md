# Chapter 7: Execution Flow and Conditions

By default, GitHub Actions halts execution of a job if any step returns a non-zero exit code. Execution flow can be modified using the `if` conditional and status check functions.

## The `if` Conditional
The `if` keyword evaluates an expression to determine if a job or step should execute. If the expression resolves to `false`, the execution is bypassed.

```yaml
steps:
  - name: Production Deployment
    if: github.ref == 'refs/heads/main'
    run: echo "Deploying..."
```
*Note: The `${{ }}` expression syntax is optional for expressions evaluated directly after the `if` keyword.*

## Status Check Functions
GitHub provides specific functions to evaluate the success or failure states of previous steps.

### `success()`
The `success()` function returns `true` if all previous steps in the job succeeded. This is the implicit default condition for all steps.

### `failure()`
The `failure()` function returns `true` if any previous step in the job failed. It is utilized to trigger notification workflows or error-handling scripts.

```yaml
steps:
  - name: Execute Tests
    run: npm run test:all

  - name: Alert on Failure
    if: failure()
    run: ./send-alert.sh
```

**`failure()` across jobs:**
When used in a job-level `if:`, `failure()` evaluates to `true` if any job listed in `needs` has a `result` of `failure`. The downstream job must declare `needs` for this to work, and must explicitly provide the `if: failure()` condition - otherwise the job is skipped by default.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: exit 1

  notify:
    runs-on: ubuntu-latest
    needs: test
    if: failure()
    steps:
      - run: echo "Test job failed!"
```
*(Reference: `17-failure-workflow.yaml`)*

### `always()`
The `always()` function returns `true` regardless of the execution status of previous steps, including cases where the workflow was cancelled. It is commonly used for resource cleanup or final reporting steps that must run unconditionally.

```yaml
steps:
  - name: Cleanup Resources
    if: always()
    run: rm -rf /tmp/cache
```

### `cancelled()`
The `cancelled()` function returns `true` exclusively if the workflow run was manually cancelled by a user via the GitHub interface or API.

## Handling Flaky Steps (`continue-on-error`)
The `continue-on-error` property instructs the workflow engine to ignore a non-zero exit code from a step, allowing subsequent steps to execute normally without failing the job.

```yaml
steps:
  - name: Non-critical Linter
    run: npm run lint
    continue-on-error: true

  - name: Build Application
    run: npm run build
```

## Step `outcome` vs `conclusion`
Every executed step records two status fields in the `steps` context. Understanding the distinction is necessary when steps use `continue-on-error`.

| Field | Definition |
|---|---|
| `outcome` | The **actual** result of the step. Reflects what the command returned. |
| `conclusion` | The **effective** result after `continue-on-error` is applied. This is what GitHub uses to determine whether the job failed. |

**Possible values for both fields:** `success`, `failure`, `cancelled`, `skipped`

**How `continue-on-error` affects these fields:**

| Scenario | `outcome` | `conclusion` | Job status |
|---|---|---|---|
| Step succeeds | `success` | `success` | Success |
| Step fails, no `continue-on-error` | `failure` | `failure` | Failure |
| Step fails, `continue-on-error: true` | `failure` | `success` | Success |

When `continue-on-error: true` is set on a failing step, the step's `conclusion` is recorded as `success`. Because the job's status is derived from `conclusion` (not `outcome`), the job remains in a success state. As a result, any subsequent step evaluating `if: failure()` will receive `false` and will be bypassed.

To inspect the true result of a step with `continue-on-error`, read `steps.<id>.outcome` rather than `steps.<id>.conclusion`.

```yaml
steps:
  - name: Failing Step
    id: failstep
    run: exit 1
    continue-on-error: true

  # This step WILL NOT run because continue-on-error kept the job in success state
  - name: Run if failed
    if: failure()
    run: echo "This will not execute"

  # This step WILL run because the job is still in success state
  - name: Inspect actual result
    run: |
      echo "Actual result:    ${{ steps.failstep.outcome }}"
      echo "Effective result: ${{ steps.failstep.conclusion }}"
      # Output:
      # Actual result:    failure
      # Effective result: success
```
*(Reference: `08-test-conditions.yaml`, `17-failure-workflow.yaml`)*
