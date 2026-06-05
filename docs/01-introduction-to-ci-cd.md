# Chapter 1: Introduction to CI/CD and GitHub Actions

## The Problem Before Automation

Before CI/CD existed, shipping software looked like this: a developer finishes a feature, manually runs the tests on their local machine, zips up the build artifacts, copies them to a server via FTP or SCP, restarts the application process, and hopes nothing broke in the process. If two developers were working simultaneously, their changes would eventually need to be merged, often days or weeks after they diverged. That merge was called an integration and it was frequently painful. Bugs that were caught weeks later were expensive to fix because the context was gone.

This was the state of the industry through the 1990s and into the early 2000s.

## What Continuous Integration Solved

Continuous Integration is the practice of merging every developer's changes into a shared codebase frequently, ideally multiple times per day, and automatically verifying that the merged result still builds and passes tests.

The "continuous" part matters. When integration happens rarely, changes accumulate and conflicts grow. When it happens continuously, conflicts are small, caught immediately, and cheap to fix.

The automated verification matters equally. A human running tests manually before every merge is a bottleneck that slows the team and eventually gets skipped under time pressure. Automation removes the human from that loop.

CI answered the question: does this code still work after merging?

## What Continuous Deployment Added

Continuous Deployment extends CI by automatically delivering the verified code to its destination. Once the build passes and tests are green, the code is packaged and deployed to the target environment without manual intervention.

The benefit is predictability and speed. Deployments that happen automatically every time a test passes are small, frequent, and low-risk. Deployments that happen manually once a month are large, infrequent, and high-risk.

Some organizations distinguish Continuous Delivery (the code is always ready to deploy but a human decides when) from Continuous Deployment (every passing build is deployed automatically). In practice, the term CD is used for both. The distinction matters for regulated industries where a human approval step is required before production.

## Where GitHub Actions Fits

GitHub Actions is the CI/CD platform built into GitHub. It was released in 2019 and has become the default choice for repositories hosted on GitHub because it requires no external accounts, no separate service configuration, and no billing setup for public repositories.

Before GitHub Actions, teams used external services like Jenkins (self-hosted), CircleCI, Travis CI, or TeamCity. These required separate accounts, webhooks configured between GitHub and the service, and credentials stored in two places. GitHub Actions eliminates that overhead because the automation lives in the same repository as the code.

GitHub Actions can do more than CI/CD. It can automate any repeatable task triggered by a repository event: triaging issues, publishing documentation, sending notifications, managing releases, enforcing code review policies. CI/CD is the most common use case, but the platform is a general-purpose automation system.

## The Core Architecture

Understanding how the pieces connect is more useful than memorizing definitions. Here is the architecture from top to bottom.

### Workflow
A workflow is a YAML file stored in `.github/workflows/`. It defines what automation should happen, when it should happen, and how it should run. A repository can have many workflow files. Each file is independent.

```yaml
# .github/workflows/build.yml
name: Build and Test
```

### Event
An event is something that happens in the repository. When an event occurs, GitHub checks if any workflow files have subscribed to it. If so, those workflows are triggered.

Events include: pushing a commit, opening a pull request, creating a release, a scheduled time, a manual button click in the UI, or the completion of another workflow.

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

### Job
A workflow contains one or more jobs. A job is a unit of work that runs on a single machine from start to finish. Jobs run in parallel by default. If you need them to run in sequence, you declare dependencies between them.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: ...

  test:
    runs-on: ubuntu-latest
    needs: build
    steps: ...
```

### Runner
A runner is the machine that executes a job. GitHub provides managed runners running Ubuntu, Windows, and macOS. You can also register your own machines as self-hosted runners.

Every time a job runs, GitHub picks a fresh runner, sets it up, runs all the steps, then discards the machine. The next job run starts on a clean machine again. This means no leftover state from previous runs, which is what makes CI pipelines reproducible.

### Step
A job is made up of steps. Steps run sequentially within a job, on the same runner machine, sharing the same filesystem. If one step writes a file, the next step can read it.

A step is either a `run` command (arbitrary shell commands) or a `uses` reference (a pre-built action from the marketplace or your own repository).

```yaml
steps:
  - name: Check out code
    uses: actions/checkout@v4    # Pre-built action

  - name: Install dependencies
    run: npm ci                  # Shell command

  - name: Run tests
    run: npm test                # Shell command
```

### Action
An action is a reusable package of code that performs a specific task. Instead of writing 20 lines of shell script to clone a repository, authenticate with AWS, or set up a specific language version, you use an action that has already solved that problem.

Actions are published on the GitHub Marketplace. They are referenced by `owner/repo@version`. The version tag pins the action to a specific release, preventing unexpected breakage when the action author releases updates.

```yaml
- uses: actions/checkout@v4           # GitHub's official checkout action
- uses: actions/setup-node@v4         # GitHub's official Node.js setup action
- uses: aws-actions/configure-aws-credentials@v4  # AWS's official credentials action
```

## How They Connect: A Mental Model

```
Repository Event (push, pull_request, schedule...)
        |
        v
   Workflow File (.github/workflows/*.yml)
        |
        v
   One or more Jobs (run in parallel by default)
        |
        v
   Runner Machine (fresh Ubuntu/Windows/macOS VM)
        |
        v
   Steps (run sequentially on that machine)
        |
        v
   run: commands / uses: actions
```

Each job gets its own fresh machine. Steps within a job share that machine. Data from one job cannot be read by another job through the filesystem - it must be passed through artifacts or job outputs.

## A Complete Workflow, Explained Line by Line

```yaml
name: Build and Test              # The name shown in the GitHub Actions UI

on:                               # The 'on' key defines what triggers this workflow
  push:                           # Trigger: any push event
    branches:                     # Filter: only pushes to these branches
      - main

jobs:                             # All jobs are defined under this key
  build:                          # 'build' is the job ID (you choose the name)
    runs-on: ubuntu-latest        # Use GitHub's managed Ubuntu runner

    steps:                        # Steps run in order on the same machine
      - name: Clone repository    # A human-readable label for the step
        uses: actions/checkout@v4 # Runs the 'checkout' action to clone the repo
                                  # Without this, the workspace is empty

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:                     # 'with' passes inputs to the action
          node-version: 20

      - name: Install dependencies
        run: npm ci               # Shell command; npm ci is the CI-safe install

      - name: Run tests
        run: npm test             # If this exits with a non-zero code, the job fails
```

When a developer pushes to `main`, GitHub reads this file, finds a matching event, and schedules the `build` job. GitHub allocates a fresh Ubuntu runner, executes the four steps in order, and reports success or failure back to the commit.

## What Happens When a Step Fails

If any `run` command exits with a non-zero exit code (which is the Unix convention for indicating an error), the step is marked as failed. All remaining steps in that job are skipped. The job is marked as failed. The workflow run is marked as failed. GitHub marks the associated commit with a red X.

This is the mechanism that makes CI useful: a failing test causes a failing step, which causes a failing job, which marks the commit as broken, which notifies the developer before any broken code can reach production.

## The `.github/workflows/` Directory

All workflow files must be placed in `.github/workflows/` relative to the root of the repository. The filename can be anything with a `.yml` or `.yaml` extension. GitHub automatically discovers and processes all files in this directory.

```
your-repository/
  .github/
    workflows/
      build.yml           # CI pipeline
      deploy.yml          # Deployment pipeline
      labeler.yml         # Auto-labeling workflow
  src/
  package.json
  README.md
```

Each file is a separate, independent workflow. They can run simultaneously. They can reference each other through the `workflow_call` or `workflow_run` trigger mechanisms.
