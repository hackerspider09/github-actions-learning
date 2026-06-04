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
*(Reference: `17-failure-workflow.yaml`)*

### `always()`
The `always()` function returns `true` regardless of the execution status of previous steps, including cases where the workflow was cancelled. It is commonly used for resource cleanup.

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

**Interaction with Status Checks:**
If a step fails but has `continue-on-error: true` defined, its formal `conclusion` is recorded as `success`. As a result, subsequent steps evaluating `if: failure()` will evaluate to `false` and bypass execution.

*(Reference: `08-test-conditions.yaml` and `17-failure-workflow.yaml`)*
