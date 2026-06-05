# GitHub Actions Complete Reference

This repository is a self-contained reference for GitHub Actions. It is designed so that a person with no prior experience can start at Chapter 0 and build complete, production-grade CI/CD pipelines by the end, and so that an experienced engineer can jump directly to any chapter to look up syntax, behavior, or security considerations.

No external documentation is required to use this reference.

## How to Read This

**If you are new to GitHub Actions or CI/CD:** Start at Chapter 0. Read sequentially through Chapter 7. Then read Chapter 20 for a complete picture of how everything connects. Return to individual chapters when you need details.

**If you know CI/CD but are new to GitHub Actions:** Start at Chapter 1 (architecture overview) then Chapter 2 (YAML format). From there, go directly to whatever topic you need.

**If you are experienced and need syntax or behavior reference:** Use the table of contents below. Each chapter covers one topic fully, with behavior explained alongside the syntax.

## Table of Contents

| # | Chapter | What it covers |
|---|---|---|
| 0 | [YAML Primer](docs/00-yaml-primer.md) | YAML format, indentation, strings, lists, mappings, multiline blocks, type gotchas, common mistakes |
| 1 | [Introduction to CI/CD and GitHub Actions](docs/01-introduction-to-ci-cd.md) | Why CI/CD exists, how GitHub Actions fits, core architecture (Workflow, Event, Job, Runner, Step, Action), first workflow walkthrough |
| 2 | [Events and Triggers](docs/02-events-and-triggers.md) | Push with branch/path/tag filters, pull_request types table, fork PR security model, workflow_dispatch with all input types and API/CLI invocation, schedule with cron syntax table, workflow_run with conclusion filtering, repository_dispatch with client_payload, common events reference table |
| 3 | [Runners and Environments](docs/03-runners-and-environments.md) | GitHub-hosted runner OS labels, hardware specs, pre-installed software, self-hosted runners, container jobs, workspace filesystem layout |
| 4 | [Jobs, Steps, and Dependencies](docs/04-jobs-and-dependencies.md) | Sequential vs parallel execution, needs, job result values (success/failure/cancelled/skipped), passing data between steps and jobs |
| 5 | [Variables, Secrets, and Environments](docs/05-variables-secrets-and-contexts.md) | Disambiguation of the two meanings of "environment", variable scoping, $GITHUB_ENV, secrets vs vars, scoping precedence, GitHub deployment environments, protection rules, GITHUB_TOKEN |
| 6 | [Contexts, Expressions, and Functions](docs/06-expressions-and-functions.md) | Full property reference for github/runner/job/steps/needs/secrets/vars/inputs/env contexts, operators, type coercion, all built-in functions, context availability table |
| 7 | [Execution Flow and Conditions](docs/07-execution-flow-and-conditions.md) | if conditionals, success/failure/always/cancelled, failure() across jobs, continue-on-error, outcome vs conclusion with comparison table |
| 8 | [Matrix Strategies](docs/08-matrix-strategies.md) | Matrix definition, multi-dimensional matrices, float/string type gotcha, exclude-single vs exclude-combination with annotated tables, fail-fast with real failure scenario, max-parallel |
| 9 | [Artifacts and Caching](docs/09-artifacts-and-caching.md) | Artifacts vs caching distinction, upload/download with all options, retention, glob patterns, restore keys (fallback cache), cache-hit detection, language-specific cache paths, setup action auto-caching |
| 10 | [Reusable Workflows and Custom Actions](docs/10-reusable-workflows-and-actions.md) | Reusable workflow inputs/outputs/secrets, calling from another repo, secrets:inherit, composite actions with inputs and outputs, shell requirement, choosing between composite and reusable |
| 11 | [Debugging Workflows](docs/11-advanced-bash-and-tricks.md) | ACTIONS_STEP_DEBUG, ACTIONS_RUNNER_DEBUG, context inspection with toJson, $GITHUB_PATH, $GITHUB_STEP_SUMMARY, tmate SSH debugging, common failure pattern table |
| 12 | [String Operations](docs/12-string-operations.md) | When to use expression functions vs Bash expansion, replace(), format(), join(), substring extraction, search and replace, case conversion, default values, prefix/suffix trimming, string length, real Docker tag pipeline example |
| 13 | [Default Environment Variables](docs/13-default-environment-variables.md) | Complete reference tables for all built-in variables (identity, git, workflow/run, paths, runner), special environment files table, practical examples |
| 14 | [Concurrency and Timeouts](docs/14-concurrency-and-timeouts.md) | concurrency groups, cancel-in-progress, dynamic group names, job-level concurrency, timeout-minutes at job and step level |
| 15 | [Permissions and the GITHUB_TOKEN](docs/15-permissions-and-github-token.md) | GITHUB_TOKEN lifecycle, permissions block at workflow and job scope, all permission scopes, OIDC keyless cloud authentication |
| 16 | [Service Containers](docs/16-service-containers.md) | Database sidecars for integration tests, health checks, port mapping, localhost vs container hostname, multiple services, private registry credentials |
| 17 | [Job and Step Outputs - Complete Reference](docs/17-job-and-step-outputs-deep-dive.md) | Output file architecture, step outputs with multiline support, outcome vs conclusion, job result values, downstream consumption, data flow summary |
| 18 | [Security Hardening](docs/18-security-hardening.md) | SHA pinning, pull_request_target risks, least-privilege permissions, script injection with safe pattern, third-party action vetting, secrets best practices, self-hosted runner security |
| 19 | [Custom Actions](docs/19-custom-actions.md) | action.yml spec, Docker container actions (Dockerfile, entrypoint), JavaScript actions (index.js, @actions/core full API, TypeScript with ncc), publishing to Marketplace |
| 20 | [Real-World CI/CD Pipeline](docs/20-real-world-pipeline.md) | End-to-end pipeline for a Node.js API: CI (lint, unit tests, integration tests with PostgreSQL, Docker build check), CD (image build/push, OIDC AWS auth, staging auto-deploy, production with required reviewers, failure notifications), all design decisions explained |
| 21 | [DevSecOps Pipeline](docs/21-devsecops-pipeline.md) | What DevSecOps is and why it exists, secrets scanning with Gitleaks (full Git history, fetch-depth:0), SAST with CodeQL (how it intercepts the build, multi-language matrix), SCA with Trivy filesystem scan, Docker image scanning with Trivy, SSH deployment to EC2 with appleboy actions, docker compose deployment pattern, required GitHub secrets/variables/environment configuration, security properties of the pipeline |

## Code Reference

The `.github/workflows/` directory contains working YAML files demonstrating specific concepts. Where a doc chapter references one of these files, it is noted inline.

| File | Concept demonstrated |
|---|---|
| `01-test-workspace.yaml` | OS validation and workspace layout |
| `02-test-env.yaml`, `03-test-github-env.yml` | Variable scoping, $GITHUB_ENV |
| `04-test-step-output.yml`, `05-test-job-output.yaml` | Step outputs, job outputs, needs context |
| `06-test-inputs.yaml` | workflow_dispatch with typed inputs |
| `07-test-context.yaml` | Reading and printing context objects |
| `08-test-conditions.yaml`, `09-test-needs-if.yaml` | if conditionals, outcome vs conclusion, job-level if with needs |
| `10-test_summary.yaml` | $GITHUB_STEP_SUMMARY markdown rendering |
| `11-secret-var-environmet.yaml` | Secrets and variables at repo and environment scope, deployment URL |
| `12-strategy-matrix.yaml` | Matrix strategies, exclude variants, fail-fast with error scenario |
| `13-cronjob.yaml` | Scheduled workflow with cron syntax |
| `14-workflow-input-output-and-reuable.yaml`, `14.1-sub-workflow.yaml` | Reusable workflow caller and callee |
| `15-workflow-run-trigger.yaml`, `15.1-success-fail-workflow.yaml` | workflow_run trigger and conclusion filtering |
| `16-artifact.yaml` | Artifact upload and download between jobs |
| `17-failure-workflow.yaml` | failure() in step and job scope, continue-on-error behavior |
