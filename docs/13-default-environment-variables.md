# Chapter 13: Default Environment Variables

GitHub Actions natively provisions a set of default environment variables for every runner execution. These variables provide critical metadata regarding the workflow run, the repository, and the triggering event. 

Default environment variables can be accessed directly within shell scripts running on the runner without requiring expression syntax (`${{ }}`).

## 1. Contexts vs. Default Environment Variables

While contexts (e.g., `${{ github.sha }}`) and default environment variables (e.g., `$GITHUB_SHA`) often contain identical data, they are evaluated at different stages of the workflow lifecycle.

- **Contexts (`${{ github.ref }}`)**: Interpolated by the GitHub Actions engine *before* the job is sent to the runner. Required for `if` conditionals and workflow definitions.
- **Default Environment Variables (`$GITHUB_REF`)**: Injected directly into the runner's shell environment. Required for native bash scripts within `run` steps to prevent script injection vulnerabilities.

## 2. Essential Default Variables

The following variables are frequently utilized within CI/CD pipelines for deployment tagging, conditional routing, and artifact naming.

### Identity and Repository Data
- **`CI`**: Always evaluates to `true`. Useful for frameworks (like React or Jest) that alter their behavior when executed in a CI environment.
- **`GITHUB_ACTOR`**: The username of the account that initiated the workflow run (e.g., `octocat`).
- **`GITHUB_REPOSITORY`**: The owner and repository name combined (e.g., `octocat/Hello-World`).
- **`GITHUB_WORKSPACE`**: The absolute path to the default working directory on the runner. This is the directory where `actions/checkout` clones the repository.

### Git and Commit Metadata
- **`GITHUB_SHA`**: The full commit SHA that triggered the workflow. Commonly used for tagging Docker images or generating unique artifact identifiers.
- **`GITHUB_REF`**: The fully qualified branch or tag reference that triggered the workflow run (e.g., `refs/heads/main` or `refs/tags/v1.0.0`).
- **`GITHUB_REF_NAME`**: The short ref name of the branch or tag that triggered the workflow run (e.g., `main` or `v1.0.0`).

### Pull Request Metadata
*Note: These variables are only populated when the workflow is triggered by a `pull_request` event.*
- **`GITHUB_HEAD_REF`**: The source branch of the pull request.
- **`GITHUB_BASE_REF`**: The target branch of the pull request.

### Execution Metadata
- **`GITHUB_WORKFLOW`**: The name of the workflow currently executing.
- **`GITHUB_RUN_ID`**: A unique numerical identifier for the specific execution of the workflow. This ID does not reset.
- **`GITHUB_RUN_NUMBER`**: A sequential number indicating how many times the specific workflow has executed. It begins at 1 and increments with each execution.
- **`GITHUB_EVENT_NAME`**: The name of the event that triggered the workflow (e.g., `push`, `pull_request`, `schedule`).

## 3. Special Environment Files

GitHub Actions uses specialized environment files to facilitate communication between steps. Rather than reading from these variables, steps append data to the files located at the paths specified by these variables.

- **`GITHUB_ENV`**: Appending `KEY=VALUE` to this file persists an environment variable for all subsequent steps within the current job.
- **`GITHUB_OUTPUT`**: Appending `KEY=VALUE` to this file registers a step output that can be referenced by subsequent steps or jobs.
- **`GITHUB_PATH`**: Appending a directory path to this file ensures the directory is added to the system `PATH` for all subsequent steps.
- **`GITHUB_STEP_SUMMARY`**: Appending Markdown content to this file renders a custom UI summary on the workflow execution page.

## Example Implementation

The following example demonstrates utilizing default environment variables natively within a Bash script.

```yaml
steps:
  - name: Build Docker Image
    run: |
      IMAGE_NAME="my-registry.com/$GITHUB_REPOSITORY"
      IMAGE_TAG="$GITHUB_REF_NAME-$GITHUB_SHA"
      
      echo "Building $IMAGE_NAME:$IMAGE_TAG"
      echo "Triggered by user: $GITHUB_ACTOR"
      
      docker build -t "$IMAGE_NAME:$IMAGE_TAG" "$GITHUB_WORKSPACE"
```
