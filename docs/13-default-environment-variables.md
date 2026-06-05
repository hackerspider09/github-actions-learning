# Chapter 13: Default Environment Variables

GitHub Actions automatically injects a set of environment variables into every runner execution. These variables are available to every `run:` step in every job without any configuration. They describe the workflow run, the repository, the triggering commit, and the runner itself.

## Contexts vs Default Environment Variables

Both mechanisms often contain the same data, but they are evaluated at different points and used in different locations.

**Contexts (`${{ github.sha }}`)** are evaluated by the GitHub Actions engine before the job is dispatched to the runner. They must be used in `if:` conditions, `name:` fields, and any other location that is part of the workflow definition rather than the shell script.

**Default environment variables (`$GITHUB_SHA`)** are injected into the runner's shell environment when the job starts. They are used inside `run:` blocks. Using them in shell scripts avoids script injection vulnerabilities because the shell reads them as variable values, not as commands.

```yaml
# Use context in the workflow definition (if condition)
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      # Use environment variable in the shell script
      - run: echo "Deploying commit $GITHUB_SHA on branch $GITHUB_REF_NAME"
```

## Complete Variable Reference

### Repository and Identity

| Variable | Example Value | Description |
|---|---|---|
| `CI` | `true` | Always `true`. Many test frameworks (Jest, pytest) modify their output format when this is set. |
| `GITHUB_ACTOR` | `octocat` | Username of the account that triggered the workflow. |
| `GITHUB_ACTOR_ID` | `1234567` | Numeric ID of the triggering account. |
| `GITHUB_REPOSITORY` | `octocat/Hello-World` | Owner and repository name combined. Used for Docker image registry paths. |
| `GITHUB_REPOSITORY_ID` | `1296269` | Numeric ID of the repository. |
| `GITHUB_REPOSITORY_OWNER` | `octocat` | Repository owner name only. |
| `GITHUB_SERVER_URL` | `https://github.com` | URL of the GitHub server. |
| `GITHUB_API_URL` | `https://api.github.com` | URL of the GitHub API. |
| `GITHUB_GRAPHQL_URL` | `https://api.github.com/graphql` | URL of the GitHub GraphQL API. |

### Git and Commit Metadata

| Variable | Example Value | Description |
|---|---|---|
| `GITHUB_SHA` | `ffac537e6cbbf9...` | Full 40-character SHA of the commit that triggered the run. Used for Docker image tagging and unique artifact identifiers. |
| `GITHUB_REF` | `refs/heads/main` | Fully qualified ref. For branches: `refs/heads/<name>`. For tags: `refs/tags/<name>`. For PRs: `refs/pull/<number>/merge`. |
| `GITHUB_REF_NAME` | `main` | Short name of the branch or tag. `main`, `v1.0.0`, `feature/login`. |
| `GITHUB_REF_TYPE` | `branch` | Either `branch` or `tag`. |
| `GITHUB_HEAD_REF` | `feature/login` | Source branch of a pull request. Empty for non-PR triggers. |
| `GITHUB_BASE_REF` | `main` | Target branch of a pull request. Empty for non-PR triggers. |

### Workflow and Run Metadata

| Variable | Example Value | Description |
|---|---|---|
| `GITHUB_WORKFLOW` | `Deploy to Production` | The `name:` of the workflow file currently executing. |
| `GITHUB_WORKFLOW_REF` | `octocat/Hello-World/.github/workflows/deploy.yml@refs/heads/main` | Full ref path of the workflow file. |
| `GITHUB_JOB` | `build` | ID of the current job (not the display name). |
| `GITHUB_ACTION` | `__run` | Name of the current action. For `run:` steps this is `__run`. |
| `GITHUB_RUN_ID` | `1658821493` | Unique ID for this specific workflow run. Persists across retries. Never resets. Used to construct run URLs. |
| `GITHUB_RUN_NUMBER` | `42` | Sequential counter for how many times this workflow has run. Starts at 1, increments with every run. |
| `GITHUB_RUN_ATTEMPT` | `1` | Which attempt number this is. Increments when a run is retried. |
| `GITHUB_EVENT_NAME` | `push` | Name of the event that triggered the run: `push`, `pull_request`, `workflow_dispatch`, `schedule`, etc. |
| `GITHUB_TRIGGERING_ACTOR` | `octocat` | Username of the account that triggered the re-run. May differ from `GITHUB_ACTOR` when a different user retries a workflow. |

### Paths and Workspace

| Variable | Example Value | Description |
|---|---|---|
| `GITHUB_WORKSPACE` | `/home/runner/work/Hello-World/Hello-World` | Absolute path to the directory where `actions/checkout` clones the repository. This is the working directory for `run:` steps by default. |
| `GITHUB_ACTION_PATH` | `/home/runner/work/_actions/my-action/v1` | Path where a custom action's files are located. Only meaningful inside an action's own steps. |
| `RUNNER_TEMP` | `/home/runner/work/_temp` | Temporary directory for the current job. Contents are deleted at the end of the job. |
| `RUNNER_TOOL_CACHE` | `/opt/hostedtoolcache` | Directory where setup actions install language runtimes (Node.js, Python, Java, etc.). |

### Runner Information

| Variable | Example Value | Description |
|---|---|---|
| `RUNNER_NAME` | `GitHub Actions 2` | Name of the runner executing the job. |
| `RUNNER_OS` | `Linux` | Operating system of the runner: `Linux`, `Windows`, or `macOS`. |
| `RUNNER_ARCH` | `X64` | CPU architecture: `X86`, `X64`, `ARM`, or `ARM64`. |
| `RUNNER_DEBUG` | `1` | Set to `1` when runner diagnostic logging is enabled via `ACTIONS_RUNNER_DEBUG`. Otherwise absent. |

## Special Environment Files

These variables point to files on the runner filesystem. Steps communicate state to the runner process by appending to these files. The runner reads them after each step and updates the workflow context accordingly.

| Variable | Usage | Effect |
|---|---|---|
| `GITHUB_ENV` | `echo "KEY=VALUE" >> $GITHUB_ENV` | Makes `KEY` available as an environment variable in all subsequent steps. |
| `GITHUB_OUTPUT` | `echo "key=value" >> $GITHUB_OUTPUT` | Registers a named output on the step, accessible via `steps.<id>.outputs.key` and mappable to job outputs. |
| `GITHUB_PATH` | `echo "/path/to/dir" >> $GITHUB_PATH` | Prepends the directory to `$PATH` for all subsequent steps. |
| `GITHUB_STEP_SUMMARY` | `echo "## Report" >> $GITHUB_STEP_SUMMARY` | Renders Markdown on the workflow run summary page. |

## Practical Examples

### Building a Docker Image with Automatic Tagging

```yaml
steps:
  - name: Build and tag Docker image
    run: |
      IMAGE="ghcr.io/$GITHUB_REPOSITORY"
      # Lowercase required by container registries
      IMAGE="${IMAGE,,}"

      # Short SHA for human-readable tags
      SHORT_SHA="${GITHUB_SHA:0:7}"

      # Build with both a SHA tag and a branch tag
      docker build \
        -t "${IMAGE}:${SHORT_SHA}" \
        -t "${IMAGE}:${GITHUB_REF_NAME//\//-}" \
        "$GITHUB_WORKSPACE"
      
      echo "Built image: ${IMAGE}:${SHORT_SHA}"
```

### Routing Logic Based on Event Type

```yaml
steps:
  - name: Log trigger information
    run: |
      echo "Workflow:     $GITHUB_WORKFLOW"
      echo "Triggered by: $GITHUB_EVENT_NAME"
      echo "Actor:        $GITHUB_ACTOR"
      echo "Branch/Tag:   $GITHUB_REF_NAME"
      echo "Commit:       $GITHUB_SHA"
      echo "Run number:   $GITHUB_RUN_NUMBER"
      echo "Run attempt:  $GITHUB_RUN_ATTEMPT"

  - name: Handle scheduled vs push differently
    run: |
      if [[ "$GITHUB_EVENT_NAME" == "schedule" ]]; then
        echo "Running full nightly test suite"
        npm run test:full
      else
        echo "Running quick test suite for commit $GITHUB_SHA"
        npm run test:quick
      fi
```

### Generating Unique Artifact Names

```yaml
steps:
  - name: Upload build artifact with unique name
    uses: actions/upload-artifact@v4
    with:
      # Unique per run, human readable
      name: build-${{ env.GITHUB_REF_NAME }}-${{ env.GITHUB_RUN_NUMBER }}
      path: ./dist/
```
