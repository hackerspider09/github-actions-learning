# Chapter 6: Contexts, Expressions, and Built-in Functions

## What Expressions Are

Expressions are pieces of logic embedded inside workflow YAML values. They are enclosed in `${{ }}` and evaluated by the GitHub Actions engine before the YAML is sent to the runner. The result replaces the `${{ }}` block in the value.

```yaml
# Before evaluation
run: echo "Triggered by ${{ github.actor }}"

# After evaluation (what the runner actually receives)
run: echo "Triggered by octocat"
```

Expressions can appear in most YAML values: step names, `if` conditions, `env` values, `with` inputs, `run` commands, and more.

## What Contexts Are

A context is an object made available by GitHub that contains information about the current workflow run. You access a context's properties using dot notation inside an expression.

```yaml
${{ github.actor }}       # The username who triggered the run
${{ runner.os }}          # The operating system of the runner
${{ job.status }}         # The current status of the job
```

Contexts are the primary way to read dynamic information about the run environment, the triggering event, and the results of previous steps.

## Full Context Reference

### The `github` Context

The `github` context contains information about the workflow run and the event that triggered it. This is the most frequently used context.

| Property | Type | Description |
|---|---|---|
| `github.actor` | string | Username of the account that triggered the run |
| `github.event_name` | string | Name of the triggering event (e.g., `push`, `pull_request`) |
| `github.repository` | string | Owner and repo name (e.g., `octocat/Hello-World`) |
| `github.repository_owner` | string | Owner name only (e.g., `octocat`) |
| `github.ref` | string | Full ref that triggered the run (e.g., `refs/heads/main`) |
| `github.ref_name` | string | Short branch or tag name (e.g., `main`, `v1.0.0`) |
| `github.sha` | string | Full 40-character commit SHA |
| `github.run_id` | number | Unique ID for this workflow run (never resets) |
| `github.run_number` | number | Sequential number for this workflow (resets per workflow) |
| `github.run_attempt` | number | Attempt number for this run (1 for first attempt, 2 for retry) |
| `github.workflow` | string | Name of the workflow |
| `github.job` | string | ID of the current job |
| `github.workspace` | string | Absolute path to the workspace directory on the runner |
| `github.event` | object | The full event payload (structure varies by event type) |
| `github.token` | string | The GITHUB_TOKEN for this run (same as `secrets.GITHUB_TOKEN`) |
| `github.server_url` | string | URL of the GitHub server (e.g., `https://github.com`) |
| `github.api_url` | string | URL of the GitHub API (e.g., `https://api.github.com`) |

**Accessing event payload properties:**
```yaml
# For a push event
${{ github.event.head_commit.message }}   # The commit message
${{ github.event.head_commit.author.name }} # Commit author

# For a pull_request event
${{ github.event.pull_request.number }}   # PR number
${{ github.event.pull_request.title }}    # PR title
${{ github.event.pull_request.user.login }} # PR author

# For a workflow_dispatch event
${{ github.event.inputs.my_input }}       # Manual input value
```

### The `runner` Context

The `runner` context contains information about the machine executing the current job.

| Property | Type | Description |
|---|---|---|
| `runner.os` | string | Operating system: `Linux`, `Windows`, or `macOS` |
| `runner.arch` | string | CPU architecture: `X86`, `X64`, `ARM`, or `ARM64` |
| `runner.name` | string | Name of the runner |
| `runner.tool_cache` | string | Path to the directory where setup actions install tools |
| `runner.temp` | string | Path to a temporary directory, cleaned after each job |
| `runner.workspace` | string | Path to the workspace (parent of GITHUB_WORKSPACE) |

```yaml
steps:
  - run: echo "Running on ${{ runner.os }} (${{ runner.arch }})"
  - run: echo "Tool cache at ${{ runner.tool_cache }}"
```

### The `job` Context

The `job` context contains information about the currently running job.

| Property | Type | Description |
|---|---|---|
| `job.status` | string | Current job status: `success`, `failure`, or `cancelled` |
| `job.container.id` | string | Container ID (only available in container jobs) |
| `job.services.<id>.id` | string | ID of a service container |
| `job.services.<id>.ports` | object | Port mappings for a service container |

```yaml
steps:
  - name: Print job status
    run: echo "${{ toJson(job) }}"
```

### The `steps` Context

The `steps` context holds the outputs and execution results of previously completed steps within the same job. Only steps that have an `id` are accessible.

| Property | Type | Description |
|---|---|---|
| `steps.<id>.outputs.<name>` | string | A named output written to `$GITHUB_OUTPUT` |
| `steps.<id>.outcome` | string | Actual result: `success`, `failure`, `cancelled`, `skipped` |
| `steps.<id>.conclusion` | string | Effective result after `continue-on-error` processing |

```yaml
steps:
  - id: build
    run: echo "result=success" >> $GITHUB_OUTPUT

  - run: echo "Build result: ${{ steps.build.outputs.result }}"
  - run: echo "Build outcome: ${{ steps.build.outcome }}"
```

### The `needs` Context

The `needs` context is available to jobs that declare dependencies. It contains the outputs and results of upstream jobs.

| Property | Type | Description |
|---|---|---|
| `needs.<job-id>.outputs.<name>` | string | A job output value from the upstream job |
| `needs.<job-id>.result` | string | Result of the upstream job: `success`, `failure`, `cancelled`, `skipped` |

```yaml
jobs:
  job2:
    needs: job1
    steps:
      - run: |
          echo "Job1 result: ${{ needs.job1.result }}"
          echo "Job1 output: ${{ needs.job1.outputs.my_value }}"
```

### The `secrets` Context

Provides access to repository, organization, and environment secrets.

```yaml
${{ secrets.MY_SECRET }}
${{ secrets.GITHUB_TOKEN }}
```

Secrets are masked in logs. Their values cannot be retrieved once set in the GitHub UI.

### The `vars` Context

Provides access to repository, organization, and environment configuration variables (non-sensitive values).

```yaml
${{ vars.APP_VERSION }}
${{ vars.TARGET_REGION }}
```

### The `inputs` Context

Available when a workflow is triggered by `workflow_dispatch` or called via `workflow_call`. Contains the values passed as inputs.

```yaml
${{ inputs.environment }}
${{ inputs.debug_mode }}
```

### The `env` Context

Provides access to environment variables defined in the workflow YAML. Note that environment variables defined via `$GITHUB_ENV` are not accessible through this context; use native shell variable syntax for those.

```yaml
env:
  MY_VAR: "hello"

steps:
  - if: ${{ env.MY_VAR == 'hello' }}    # Use env context in if conditions
    run: echo "$MY_VAR world"           # Use shell variable in run steps
```

## Operators in Expressions

Expressions support standard comparison and logical operators.

| Operator | Description | Example |
|---|---|---|
| `==` | Equal | `github.actor == 'octocat'` |
| `!=` | Not equal | `github.event_name != 'schedule'` |
| `<` | Less than | `github.run_number < 100` |
| `>` | Greater than | `github.run_number > 1` |
| `<=` | Less than or equal | `needs.build.result <= 'success'` |
| `>=` | Greater than or equal | Numbers only |
| `&&` | Logical AND | `github.ref == 'refs/heads/main' && github.event_name == 'push'` |
| `||` | Logical OR | `github.event_name == 'push' || github.event_name == 'workflow_dispatch'` |
| `!` | Logical NOT | `!contains(github.event.head_commit.message, 'skip ci')` |

### Type Coercion in Comparisons
GitHub Actions automatically converts types in comparisons. A string `"true"` equals the boolean `true`. A null value equals an empty string. Numeric strings are compared numerically. This can produce unexpected results if you compare against a value that might be null.

## Built-in Functions

### `contains(haystack, needle)`
Returns `true` if the haystack string or array contains the needle.

```yaml
# String search
if: contains(github.event.head_commit.message, '[skip ci]')

# Array search
if: contains(github.event.commits.*.message, 'HOTFIX')
```

### `startsWith(string, prefix)` and `endsWith(string, suffix)`
```yaml
if: startsWith(github.ref, 'refs/tags/v')
if: endsWith(github.repository, '-backend')
```

### `format(template, arg0, arg1, ...)`
Replaces `{0}`, `{1}`, etc. placeholders in a template string.

```yaml
env:
  PR_LABEL: ${{ format('pr-{0}-{1}', github.event.pull_request.number, github.actor) }}
  # Result: pr-42-octocat
```

### `join(array, separator)`
Concatenates array elements with a separator string.

```yaml
env:
  COMMIT_LIST: ${{ join(github.event.commits.*.id, ', ') }}
```

### `toJson(value)`
Converts any value to a formatted JSON string. Primarily used for debugging to inspect the full contents of a context object.

```yaml
steps:
  - name: Inspect github context
    run: echo '${{ toJson(github) }}'

  - name: Inspect needs context
    run: echo '${{ toJson(needs) }}'

  - name: Inspect job context
    run: echo '${{ toJson(job) }}'
```

This is one of the most useful debugging tools. When a workflow behaves unexpectedly, printing the full context reveals exactly what values are available.

### `fromJson(string)`
Parses a JSON string into an object or array. This is required when data has been serialized as a string to pass through job outputs (which only support string values).

```yaml
jobs:
  prepare:
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: echo 'matrix=["node-18","node-20","node-22"]' >> $GITHUB_OUTPUT

  build:
    needs: prepare
    strategy:
      matrix:
        version: ${{ fromJson(needs.prepare.outputs.matrix) }}
    steps:
      - run: echo "Building for ${{ matrix.version }}"
```

### `hashFiles(path)`
Generates a SHA-256 hash of the files matching the given path pattern. Multiple patterns can be provided as additional arguments. This is used to generate cache keys that automatically invalidate when dependencies change.

```yaml
key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
key: ${{ runner.os }}-python-${{ hashFiles('**/requirements.txt', '**/setup.py') }}
```

## Context Availability by Workflow Location

Not all contexts are available in all parts of a workflow. Using a context where it is not available returns an empty value or causes an error.

| Context | Workflow level | Job level | Step level |
|---|---|---|---|
| `github` | Yes | Yes | Yes |
| `env` | Yes | Yes | Yes |
| `vars` | Yes | Yes | Yes |
| `secrets` | No | Yes | Yes |
| `runner` | No | Yes | Yes |
| `job` | No | No | Yes |
| `steps` | No | No | Yes |
| `needs` | No | Yes | Yes |
| `inputs` | Yes | Yes | Yes |
