# Chapter 3: Runners and Environments

A runner is the server that executes a job. Every job declaration must specify which runner to use via the `runs-on` key. Without a runner, the job has nowhere to execute.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
```

## 1. GitHub-Hosted Runners

GitHub maintains a fleet of virtual machines that are provisioned on demand for every job. When a job is triggered, GitHub allocates a fresh machine, sets it up, runs the job, then destroys the machine completely. The next run starts on a completely clean machine with no memory of previous runs.

This ephemeral model has an important consequence: anything your job needs that is not pre-installed must be installed during the job itself, and anything your job produces that you want to keep after the job ends must be explicitly saved using artifacts or caching (covered in Chapter 9).

### Available Operating Systems

| Label | Operating System | Common Use |
|---|---|---|
| `ubuntu-latest` | Ubuntu 24.04 LTS | General Linux workloads, Docker, most CI tasks |
| `ubuntu-24.04` | Ubuntu 24.04 LTS | Explicit version pin for reproducibility |
| `ubuntu-22.04` | Ubuntu 22.04 LTS | Workloads requiring an older Ubuntu version |
| `ubuntu-20.04` | Ubuntu 20.04 LTS | Legacy compatibility |
| `windows-latest` | Windows Server 2022 | Windows-specific builds, .NET Framework, MSBuild |
| `windows-2022` | Windows Server 2022 | Explicit version pin |
| `macos-latest` | macOS 14 (Sonoma) | iOS and macOS app builds using Xcode |
| `macos-14` | macOS 14 (Sonoma) | Apple Silicon (M-series chip) runner |
| `macos-13` | macOS 13 (Ventura) | Intel chip runner |

The `latest` labels always point to the most recent stable version GitHub supports. Using `latest` means your workflow automatically uses a newer OS when GitHub updates the label, which can occasionally cause breakage if your workflow relies on a specific OS behavior. For maximum reproducibility in production pipelines, pin to a specific version label.

### Hardware Specifications (Standard Runners)

| Resource | Ubuntu / Windows | macOS |
|---|---|---|
| CPU cores | 4 | 3 |
| RAM | 16 GB | 7 GB |
| SSD storage | 14 GB | 14 GB |

GitHub also offers larger runners with more CPU and RAM for repositories on paid plans. These are configured in the repository or organization settings and referenced by custom labels.

### Pre-installed Software

GitHub-hosted runners come with a substantial amount of software pre-installed. This reduces setup time for common workflows significantly.

**Always available on Ubuntu runners:**

| Category | Software |
|---|---|
| Version control | Git, Git LFS |
| Containers | Docker, Docker Compose, Podman |
| Languages | Python 3, Node.js, Ruby, Java (multiple versions), Go, .NET |
| Package managers | npm, pip, gem, Maven, Gradle, Homebrew |
| Cloud CLIs | AWS CLI, Azure CLI, Google Cloud CLI |
| Build tools | Make, CMake, GCC, Clang |
| Infrastructure | Terraform, kubectl, Helm |
| Utilities | curl, wget, jq, unzip, tar, zip, openssl |

The full list of pre-installed software for each runner image is published in the `actions/runner-images` repository on GitHub. It is updated regularly and specific software versions can be checked there.

### When Pre-installed Software Is Not Enough

If you need a specific version not pre-installed, use a setup action before your build steps:

```yaml
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: '21'

  - uses: actions/setup-python@v5
    with:
      python-version: '3.12'

  - uses: actions/setup-java@v4
    with:
      distribution: 'temurin'
      java-version: '21'

  - uses: actions/setup-go@v5
    with:
      go-version: '1.22'
```

Setup actions handle downloading the runtime, configuring `PATH`, and making the version available to all subsequent steps.

## 2. Self-Hosted Runners

A self-hosted runner is any machine you register with GitHub to run jobs. It can be a physical server, a VM, a cloud instance, a Raspberry Pi, or a container. You install the GitHub Actions runner software on the machine, authenticate it to your repository or organization, and GitHub routes jobs to it.

```yaml
jobs:
  build:
    runs-on: self-hosted    # Route this job to any registered self-hosted runner
```

### Why Use Self-Hosted Runners

**Access to private infrastructure:** GitHub-hosted runners run in GitHub's cloud. They cannot reach internal databases, private APIs, or on-premises services unless those services are exposed to the internet. A self-hosted runner inside your network can reach everything the machine can reach.

**Specific hardware requirements:** GPU workloads for machine learning, builds requiring ARM processors, or builds that need terabytes of fast local storage require specific hardware that GitHub's standard runners do not provide.

**Cost control:** GitHub charges by the minute for GitHub-hosted runner usage (free minutes are included per plan). A long-running test suite on a self-hosted runner uses no GitHub minutes.

**Persistent state:** GitHub-hosted runners are destroyed after each job. Self-hosted runners persist between jobs. This means build caches, downloaded dependencies, and installed tools remain between runs, which can significantly speed up builds.

### Risks of Self-Hosted Runners

**Security risk on public repositories:** If your repository is public and you run a self-hosted runner, any person can fork your repository, open a pull request, and potentially trigger jobs that run code on your machine. Never use self-hosted runners on public repositories without careful protection rules that limit which events and which branches can trigger jobs.

**Maintenance:** You are responsible for keeping the runner machine patched, scaled, and monitored. GitHub-hosted runners have this handled for you.

### Runner Labels

Labels allow you to route jobs to specific runners based on capabilities. A self-hosted runner can have multiple labels.

```yaml
# Route to any self-hosted runner
runs-on: self-hosted

# Route to a self-hosted runner with specific labels
runs-on: [self-hosted, linux, gpu, high-memory]
```

Labels are assigned when you register the runner. You can add custom labels through the repository settings to represent capabilities like `gpu`, `arm64`, or `production-only`.

### Runner Groups

Runner groups are an organization-level feature that controls which repositories can access which self-hosted runners. This is the mechanism for multi-team organizations where some runners are shared infrastructure and others are team-specific.

## 3. Container Jobs

A job can run all of its steps inside a Docker container rather than directly on the runner's operating system. This is useful when your job needs a specific environment that is more easily expressed as a Docker image than as a set of setup steps.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: node:20-alpine
      credentials:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

When `container:` is specified, GitHub starts the Docker container on the runner and executes all steps inside it. The runner machine itself acts as the host that runs Docker.

## 4. The Runner Workspace

When a GitHub-hosted runner is provisioned, it starts with an empty workspace. The `GITHUB_WORKSPACE` environment variable points to this directory. Its value is typically `/home/runner/work/<repo-name>/<repo-name>` on Ubuntu.

The repository code is not cloned automatically. The workspace is empty until you explicitly clone it using `actions/checkout`.

```yaml
steps:
  - name: Workspace is empty here
    run: ls -la   # Shows an empty directory or missing directory

  - name: Clone the repository
    uses: actions/checkout@v4

  - name: Workspace now has code
    run: ls -la   # Shows all repository files
```

This design is intentional. Some jobs do not need the repository code. A notification job, a release job that downloads pre-built artifacts, or a job that only interacts with external APIs may not need to clone the code at all. Cloning is opt-in.

### The Workspace Filesystem Layout

```
/home/runner/
  work/
    <repository-name>/
      <repository-name>/    <-- GITHUB_WORKSPACE points here (your code after checkout)
  _tool/                    <-- Setup actions install tools here (Node, Python, etc.)
  _temp/                    <-- Temporary files used by actions
```

Understanding this layout matters when you are writing paths in your workflow. Always use `$GITHUB_WORKSPACE` or relative paths from the workspace root rather than hardcoded absolute paths, because the exact path varies between operating systems and runner configurations.
