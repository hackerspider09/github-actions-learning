# Chapter 4: Jobs, Steps, and Dependencies

A workflow consists of jobs, and jobs consist of steps. Understanding execution order and data transfer between these components is necessary for workflow configuration.

## Steps (Sequential Execution)
Steps within a single job execute sequentially. A subsequent step will not begin until the preceding step completes. If a step fails, the remaining steps in the job are bypassed by default.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Step 1
        run: echo "Executed first"
        
      - name: Step 2
        run: echo "Executed second"
```

## Jobs (Parallel Execution)
By default, multiple jobs within a workflow execute in parallel on separate runners.

```yaml
jobs:
  job-a:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Executes in parallel with job-b"
      
  job-b:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Executes in parallel with job-a"
```

## The `needs` Keyword (Sequential Jobs)
To enforce sequential execution among jobs, define dependencies using the `needs` keyword. A dependent job will only start after the required job completes successfully.

```yaml
jobs:
  test-code:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy-code:
    runs-on: ubuntu-latest
    needs: test-code
    steps:
      - run: ./deploy.sh
```
*(Reference: `09-test-needs-if.yaml`)*

Multiple dependencies can be declared via an array:
```yaml
needs: [test-code, security-scan, build-app]
```

## Job Result Values
When a job completes, GitHub assigns it a `result` value. This result is accessible in downstream jobs via the `needs` context and is used to implement conditional logic at the job level.

The possible values for `needs.<job-id>.result` are:

| Value | Meaning |
|---|---|
| `success` | All steps in the job completed without error. |
| `failure` | At least one step exited with a non-zero code and was not suppressed by `continue-on-error`. |
| `cancelled` | The workflow run was cancelled before the job completed. |
| `skipped` | The job was skipped because its own `if:` condition evaluated to `false`. |

### Using Job Result in `if:` Conditions
A downstream job can evaluate the result of an upstream job using `needs.<job-id>.result` in an `if:` conditional. This pattern is commonly used to implement notification or rollback logic that must run even when the upstream job fails.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh

  notify-on-failure:
    runs-on: ubuntu-latest
    needs: deploy
    # The default behaviour when needs is declared is to skip this job
    # if 'deploy' fails. The if: condition overrides that default.
    if: needs.deploy.result == 'failure'
    steps:
      - run: ./send-alert.sh
```

**Important:** When a job declares `needs`, the default implicit condition for that job is `success()`. If the upstream job fails, the downstream job is skipped rather than running. To override this and execute the downstream job when the upstream has failed, an explicit `if:` condition must be provided, as shown above.

**Checking multiple upstream results:**
```yaml
jobs:
  report:
    needs: [build, test, security-scan]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Build result:    ${{ needs.build.result }}"
          echo "Test result:     ${{ needs.test.result }}"
          echo "Security result: ${{ needs.security-scan.result }}"
```
*(Reference: `09-test-needs-if.yaml`)*

## Passing Data Between Steps
Steps within the same job share the runner environment. To pass variables between steps, write the data to the `$GITHUB_OUTPUT` environment file. The step generating the output must have an `id` defined.

```yaml
steps:
  - name: Generate Value
    id: step1
    run: |
      MY_NUM=$RANDOM
      echo "my_random_number=$MY_NUM" >> $GITHUB_OUTPUT

  - name: Read Value
    run: |
      echo "The number is ${{ steps.step1.outputs.my_random_number }}"
```
*(Reference: `04-test-step-output.yml`)*

## Passing Data Between Jobs
Jobs execute on isolated runners. To pass data from one job to another, map the step output to a job-level output. The receiving job must declare a dependency on the sending job via `needs`.

```yaml
jobs:
  Job_A:
    runs-on: ubuntu-latest
    outputs:
      final_number: ${{ steps.step1.outputs.my_random_number }}
    steps:
      - name: Generate Number
        id: step1
        run: echo "my_random_number=99" >> $GITHUB_OUTPUT

  Job_B:
    runs-on: ubuntu-latest
    needs: Job_A
    steps:
      - name: Read from Job A
        run: echo "Value received: ${{ needs.Job_A.outputs.final_number }}"
```
*(Reference: `05-test-job-output.yaml`)*
