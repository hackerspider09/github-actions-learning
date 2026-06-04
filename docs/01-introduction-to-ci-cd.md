# Chapter 1: Introduction to CI/CD and GitHub Actions

## What is CI/CD?
Continuous Integration (CI) and Continuous Deployment (CD) are practices that automate the building, testing, and deployment of code.

### Continuous Integration (CI)
CI involves automatically building and testing code every time a change is committed. This process ensures that new code integrates seamlessly with the existing codebase and catches errors early in the development cycle.

### Continuous Deployment (CD)
CD extends CI by automatically deploying the integrated code to production or staging environments. This eliminates manual deployment steps, allowing software to be delivered consistently and rapidly.

## What is GitHub Actions?
GitHub Actions is a CI/CD platform integrated directly into GitHub. It enables the automation of software workflows directly from the repository without requiring external CI/CD services.

## Core Concepts
GitHub Actions relies on the following core components:

1. **Workflow**
   A configurable automated process defined by a YAML file in the `.github/workflows/` directory. A repository can contain multiple workflows.

2. **Event**
   A specific activity in a repository that triggers a workflow to run (e.g., a push, a pull request creation, or a scheduled time).

3. **Runner**
   A server that executes a specific job. Each runner executes one job at a time. GitHub provides hosted runners, or custom self-hosted runners can be configured.

4. **Job**
   A logical grouping of steps. All steps within a single job execute sequentially on the exact same runner instance, allowing them to share the local file system. Multiple jobs within a workflow execute on separate runners and run in parallel by default.

5. **Step**
   An individual task that runs commands in a job. A step can be a shell script or an action. Steps run sequentially and share the same runner environment.

6. **Action**
   A standalone command that is combined into steps to create a job. Actions are reusable units of code designed to perform specific, frequently repeated tasks.

## Workflow Structure
The following demonstrates the standard structure of a GitHub Actions YAML file:

```yaml
name: Example Workflow

on: 
  push:

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Print Status
        run: echo "Executing workflow"

      - name: Checkout Code
        uses: actions/checkout@v4
```
