# GitHub Actions TODO / Missing Concepts

This file tracks features and concepts in GitHub Actions that are not yet implemented in the `github-actions-learning` repository. 

> Note: Artifacts sharing between jobs was actually implemented in `16-artifact.yaml`! It is fully functioning. The items below are the actual missing pieces to learn next.

- [ ] **Dependency Caching (`actions/cache`)**: How to speed up builds by caching node_modules, maven dependencies, or pip packages.
- [ ] **Composite Actions**: Creating custom actions directly in this repository using `runs: using: "composite"` instead of reusable workflows to encapsulate logic.
- [ ] **Docker Container Actions**: Creating actions that run isolated inside a Docker container.
- [ ] **JavaScript/TypeScript Actions**: Building custom actions using the GitHub Actions Toolkit.
- [ ] **Service Containers**: Spinning up background services (like PostgreSQL or Redis) for the duration of a job to run integration tests against.
- [ ] **Environment Protection Rules**: Setting up environments with required manual approvals or wait timers before deployment jobs execute.
- [ ] **Matrix Advanced Features**: Using `include` to add specific combinations or variables to a matrix, and `max-parallel` to limit concurrency.
- [ ] **Concurrency**: Cancelling redundant in-progress workflow runs using `concurrency: group: ... cancel-in-progress: true`.
- [ ] **OIDC Integration (OpenID Connect)**: Authenticating securely with cloud providers (AWS, Azure, GCP) without storing long-lived credentials as secrets.
- [ ] **Job and Workflow Timeouts**: Using `timeout-minutes` to kill hanging jobs.
- [ ] **Passing Secrets to Reusable Workflows**: How to explicitly pass secrets down to `workflow_call` sub-workflows using `secrets: inherit`.
