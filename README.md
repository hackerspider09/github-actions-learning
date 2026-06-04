# GitHub Actions Documentation

This repository contains comprehensive documentation detailing GitHub Actions configuration and execution architecture. The materials progress from foundational CI/CD concepts to advanced workflow construction.

The documentation is organized sequentially and includes generic syntax explanations coupled with practical implementations drawn from the workflow files present in this repository.

## Table of Contents

Documentation files are located in the `docs/` directory:

1. [Introduction to CI/CD](docs/01-introduction-to-ci-cd.md)
   *Overview of CI/CD, GitHub Actions architecture, and core components (Workflows, Jobs, Steps).*
2. [Events and Triggers](docs/02-events-and-triggers.md)
   *Workflow invocation via code pushes, pull requests, scheduled cron expressions, and manual dispatch.*
3. [Runners and Environments](docs/03-runners-and-environments.md)
   *Execution environments, comparing GitHub-hosted and self-hosted runners, and workspace initialization.*
4. [Jobs, Steps, and Dependencies](docs/04-jobs-and-dependencies.md)
   *Parallel vs. sequential execution, job dependencies, and data transfer mechanisms via outputs.*
5. [Variables, Secrets, and Contexts](docs/05-variables-secrets-and-contexts.md)
   *Environment variable scope hierarchy, dynamic variable assignment, secret management, and target environments.*
6. [Contexts, Expressions, and Functions](docs/06-expressions-and-functions.md)
   *Expression syntax evaluation, context objects, and built-in function utilization.*
7. [Execution Flow and Conditions](docs/07-execution-flow-and-conditions.md)
   *Conditional logic implementation, error handling, status check functions, and cleanup operations.*
8. [Matrix Strategies](docs/08-matrix-strategies.md)
   *Parallel job generation across multi-dimensional variable arrays and exclusion configurations.*
9. [Artifacts and Caching](docs/09-artifacts-and-caching.md)
   *Data persistence across runner lifecycles and dependency retrieval optimization.*
10. [Reusable Workflows and Actions](docs/10-reusable-workflows-and-actions.md)
    *Modularization of pipeline components using workflow templates and custom composite actions.*
11. [Advanced Bash Operations](docs/11-advanced-bash-and-tricks.md)
    *System path modification, generating custom markdown summaries, and SSH debugging.*
12. [String Operations](docs/12-string-operations.md)
    *Comprehensive guide to string manipulation using native GitHub expression functions and Bash parameter expansion.*
13. [Default Environment Variables](docs/13-default-environment-variables.md)
    *Reference for built-in runner variables such as GITHUB_SHA, GITHUB_WORKSPACE, and GITHUB_REF.*

## Code Reference Directory

The `.github/workflows/` directory contains active YAML files demonstrating the implementation of the documented concepts:

* `01-test-workspace.yaml`: Operating system validation and workspace configuration.
* `02-test-env.yaml` & `03-test-github-env.yml`: Environment variable scoping and dynamic assignment.
* `04-test-step-output.yml` & `05-test-job-output.yaml`: Output transfer between steps and jobs.
* `06-test-inputs.yaml`: Manual workflow invocation via input parameters.
* `07-test-context.yaml`: Retrieval and formatting of standard context payloads.
* `08-test-conditions.yaml` & `09-test-needs-if.yaml`: Flow control and conditional execution.
* `10-test_summary.yaml`: Markdown generation for workflow summaries.
* `11-secret-var-environmet.yaml`: Secret management across targeted deployment environments.
* `12-strategy-matrix.yaml`: Parallel execution matrices.
* `13-cronjob.yaml`: Scheduled execution triggers.
* `14-workflow-input-output-and-reuable.yaml` & `14.1`: Invocation of modular workflows.
* `15-workflow-run-trigger.yaml` & `15.1`: Event chaining based on upstream workflow status.
* `16-artifact.yaml`: Artifact archiving and retrieval.
* `17-failure-workflow.yaml`: Error handling and execution bypass directives.

