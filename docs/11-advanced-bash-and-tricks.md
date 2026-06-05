# Chapter 11: Debugging Workflows

When a workflow behaves unexpectedly, the GitHub Actions UI shows the log output from each step. Most failures are diagnosed from logs alone. This chapter covers the tools and techniques available when logs are not enough.

## Reading Logs Effectively

Each step in the GitHub Actions UI expands to show its stdout and stderr output. When a step fails, GitHub highlights the log section for that step in red. The exit code that caused the failure is shown at the bottom of the failed step's log.

To read a log:
1. Navigate to the repository on GitHub.
2. Click the Actions tab.
3. Click the specific workflow run.
4. Click the job name in the left sidebar.
5. Click any step to expand its log output.

Failed steps are automatically expanded. Successful steps are collapsed by default.

## Enabling Debug Logging

GitHub provides two secret-controlled flags that increase the verbosity of workflow logs. These are not repository secrets in the cryptographic sense; they are repository variables or secrets that the Actions runner reads to change its behavior.

### Step Debug Logging (`ACTIONS_STEP_DEBUG`)

Setting the secret `ACTIONS_STEP_DEBUG` to `true` in your repository causes the runner to print additional information inside each step. This includes the exact commands being run, environment variable resolution, and detailed output from actions.

**How to enable:**
1. Go to **Settings → Secrets and variables → Actions → Secrets**.
2. Create a secret named `ACTIONS_STEP_DEBUG` with the value `true`.
3. Re-run the failed workflow.

The logs will now contain lines prefixed with `::debug::`, showing internal state that is hidden during normal runs.

**To enable for a single re-run without storing the secret permanently:**
When re-running a job from the GitHub UI, there is a checkbox labeled "Enable debug logging". Checking it enables step debug logging for that single re-run only.

### Runner Diagnostic Logging (`ACTIONS_RUNNER_DEBUG`)

Setting `ACTIONS_RUNNER_DEBUG` to `true` adds diagnostic logs about the runner itself: network connectivity, worker process management, and runner initialization. This is useful when jobs fail to start or when there are problems with runner registration.

Create it as a secret with value `true`, the same way as `ACTIONS_STEP_DEBUG`. Both can be enabled simultaneously.

## Inspecting Context Values

When a workflow condition does not behave as expected, print the full context to understand exactly what values are present.

```yaml
steps:
  - name: Debug github context
    run: echo '${{ toJson(github) }}'

  - name: Debug event payload
    run: echo '${{ toJson(github.event) }}'

  - name: Debug all environment variables
    run: env | sort

  - name: Debug needs context
    run: echo '${{ toJson(needs) }}'

  - name: Debug steps context
    run: echo '${{ toJson(steps) }}'
```

`toJson()` converts the context object to a formatted JSON string that can be read in the log. This is the fastest way to answer questions like: what is the exact value of `github.ref`? What did the upstream job produce in its outputs?

## Printing Environment Variables

```yaml
steps:
  - name: Show all environment variables
    run: env | sort

  - name: Show specific GitHub variables
    run: |
      echo "GITHUB_SHA:        $GITHUB_SHA"
      echo "GITHUB_REF:        $GITHUB_REF"
      echo "GITHUB_REF_NAME:   $GITHUB_REF_NAME"
      echo "GITHUB_ACTOR:      $GITHUB_ACTOR"
      echo "GITHUB_WORKSPACE:  $GITHUB_WORKSPACE"
      echo "GITHUB_EVENT_NAME: $GITHUB_EVENT_NAME"
      echo "GITHUB_RUN_ID:     $GITHUB_RUN_ID"
```

## Modifying the System Path (`$GITHUB_PATH`)

When a step installs a custom executable that needs to be available in subsequent steps without its full path, append its directory to the system PATH using the `$GITHUB_PATH` file.

```yaml
steps:
  - name: Install custom tool
    run: |
      mkdir -p "$HOME/custom-tools"
      curl -Lo "$HOME/custom-tools/mytool" https://example.com/releases/mytool-linux-amd64
      chmod +x "$HOME/custom-tools/mytool"
      echo "$HOME/custom-tools" >> $GITHUB_PATH

  - name: Use the tool by name
    run: mytool --version    # Works because the directory is now in PATH
```

Without the `$GITHUB_PATH` append, the second step would have to call the tool using its full path (`$HOME/custom-tools/mytool --version`). The `$GITHUB_PATH` mechanism persists the path change across steps; setting `export PATH=...` inside a step does not persist to the next step because each step runs in a new shell process.

## Generating Markdown Summaries (`$GITHUB_STEP_SUMMARY`)

Workflow summaries appear on the workflow run page as a rendered document. Any step can write markdown content to the `$GITHUB_STEP_SUMMARY` file, and GitHub renders it after the job completes.

This is useful for surfacing test results, coverage numbers, deployment status, or any structured data that would otherwise require reading through raw logs.

```yaml
steps:
  - name: Run tests
    id: tests
    run: npm test -- --reporter json > test-output.json
    continue-on-error: true

  - name: Generate test summary
    run: |
      PASSED=$(jq '.numPassedTests' test-output.json)
      FAILED=$(jq '.numFailedTests' test-output.json)
      TOTAL=$(jq '.numTotalTests' test-output.json)

      echo "## Test Results" >> $GITHUB_STEP_SUMMARY
      echo "" >> $GITHUB_STEP_SUMMARY
      echo "| Metric | Count |" >> $GITHUB_STEP_SUMMARY
      echo "| --- | --- |" >> $GITHUB_STEP_SUMMARY
      echo "| Total | $TOTAL |" >> $GITHUB_STEP_SUMMARY
      echo "| Passed | $PASSED |" >> $GITHUB_STEP_SUMMARY
      echo "| Failed | $FAILED |" >> $GITHUB_STEP_SUMMARY
```
*(Reference: `10-test_summary.yaml`)*

The rendered summary is visible to anyone who can see the workflow run, making it useful for reporting to non-technical stakeholders without requiring them to read log files.

## Interactive SSH Debugging with tmate

When the issue cannot be diagnosed from logs alone, a live interactive SSH session into the runner is possible using the `mxschmitt/action-tmate` community action. This pauses execution and prints an SSH connection string in the logs.

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Set up environment
    run: npm ci

  - name: Interactive debug session
    uses: mxschmitt/action-tmate@v3
    # Execution pauses here. The log shows:
    # ssh <session-id>@nyc1.tmate.io
    # Connect from your local terminal to get a shell on the runner.

  - name: Run tests
    run: npm test
```

The session stays open until you type `touch /tmp/continue` in the SSH session or the job times out. After reconnecting, you can inspect the filesystem, run commands manually, test fixes, and then signal the session to continue.

**Security:** Never merge a workflow containing `action-tmate` into a permanent branch. The session exposes the full runner environment including any secrets injected into environment variables. Use it only in temporary branches for debugging.

## Common Failure Patterns and Their Causes

| Symptom | Likely Cause |
|---|---|
| Step fails immediately with `not found` | Command not installed; check pre-installed software or add a setup action |
| `actions/checkout` succeeds but files are missing | Wrong branch checked out; check `ref:` input on checkout action |
| Secret is empty in a step | Secret not set in the correct scope (repo vs environment); check Settings |
| Job is skipped unexpectedly | `if:` condition evaluates to false; print context to check values |
| Job waits indefinitely | Environment protection rule requires manual approval; check Environments in Settings |
| Cache is never hit | Key changes every run; check that the lockfile path is correct in `hashFiles()` |
| Artifact download fails | Job declaring `needs:` was skipped; the artifact was never uploaded |
| Step fails with `permission denied` | `GITHUB_TOKEN` lacks required permission; add `permissions:` block |
