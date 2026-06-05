# Chapter 9: Artifacts and Caching

GitHub-hosted runners are destroyed after each job completes. Any file written to the runner's filesystem during the job is gone permanently when the job ends. Artifacts and caching are the two mechanisms for working around this limitation in different ways.

## The Difference Between Artifacts and Caching

These two features are often confused but serve entirely different purposes.

**Artifacts** are for outputs your workflow produces. They are the deliverable results of a job: compiled binaries, test reports, coverage files, packaged applications, deployment bundles. Artifacts persist after the workflow run ends and can be downloaded from the GitHub UI or API. They can also be passed between jobs within the same workflow run.

**Caching** is for inputs your workflow consumes repeatedly. Package dependencies like `node_modules`, pip packages, or Maven JARs do not change unless your lockfile changes. Caching saves these between workflow runs so that future runs do not re-download the same files from the internet.

The distinction matters: cache is disposable and GitHub can delete it at any time. Artifacts are preserved for a defined retention period and are treated as outputs of your workflow.

## Part 1: Artifacts

### How Artifacts Work

When you upload an artifact, GitHub compresses the specified files and stores them on its infrastructure. The artifact is associated with the specific workflow run that created it. You can then:

- Download the artifact from the GitHub UI (Actions tab, click the workflow run)
- Download it using the GitHub API
- Retrieve it in a different job within the same workflow run using the `download-artifact` action

### Uploading an Artifact

Use the `actions/upload-artifact` action to save files after they are produced.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build application
        run: npm run build    # Produces ./dist/

      - name: Run tests and generate report
        run: npm test -- --reporter=json --output-file=test-results.json
        continue-on-error: true   # Upload report even if tests fail

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: production-build          # Name used to retrieve the artifact
          path: ./dist/                   # Directory or file to upload
          retention-days: 30              # How long to keep it (default: 90)

      - name: Upload test report
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results.json
          if-no-files-found: error        # Fail if the file is missing
```

**`name`**: The label for the artifact. Must be unique within the workflow run. Used to retrieve it later.

**`path`**: File or directory to upload. Supports glob patterns. Multiple paths can be listed using a multiline string.

**`retention-days`**: How many days the artifact is kept before GitHub deletes it automatically. Range: 1 to 90 days. After the retention period expires, the artifact is permanently deleted. The repository's default retention setting applies if this field is omitted.

**`if-no-files-found`**: What to do when the path matches nothing. Options: `warn` (default), `error` (fail the step), `ignore` (silently skip).

### Downloading an Artifact in a Later Job

Artifacts are most commonly used to pass compiled outputs from a build job to a deploy job, avoiding the need to rebuild in the deploy job.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: production-build
          path: ./dist/

  deploy:
    runs-on: ubuntu-latest
    needs: build              # Must run after build job completes
    steps:
      - name: Download the build
        uses: actions/download-artifact@v4
        with:
          name: production-build
          path: ./dist/       # Where to place the downloaded files

      - name: Deploy
        run: ./deploy.sh ./dist/
```
*(Reference: `16-artifact.yaml`)*

The `needs:` declaration is required in the deploying job. Without it, there is no guarantee the build job has completed and the artifact exists.

### Downloading Without a Job Dependency

If you need to download an artifact from any workflow run (not just the current one), use the GitHub API. However, if you need to download an artifact produced in a previous run of the same workflow, the standard pattern is to use the `actions/download-artifact` action without `needs`, combined with `run-id`:

```yaml
- uses: actions/download-artifact@v4
  with:
    name: production-build
    run-id: ${{ github.event.workflow_run.id }}  # ID of the upstream run
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Uploading Multiple Files with Glob Patterns

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: test-outputs
    path: |
      test-results/**/*.xml
      coverage/lcov.info
      logs/
```

### Artifact Size Limits

Individual artifacts have a maximum size of 10 GB (compressed). If your build output exceeds this, archive it manually and upload the archive:

```yaml
- run: tar -czf build-output.tar.gz ./dist/
- uses: actions/upload-artifact@v4
  with:
    name: build-archive
    path: build-output.tar.gz
```

## Part 2: Caching

### Why Caching Exists

Every workflow run on a fresh GitHub-hosted runner must download all external dependencies from scratch. A Node.js project with 500 packages, a Python project with 80 pip packages, or a Java project using Maven must download all of them every run. For a project with many dependencies, this can add several minutes to every workflow run.

Caching stores these downloaded packages between runs. On the next run, the cache is restored before the install step. The install tool (npm, pip, Maven) finds the packages already present and skips downloading them.

### How Cache Lookup Works

Every cache entry has a key. When a job runs:
1. GitHub checks if a cache exists with the exact key.
2. If found (cache hit), the cached files are restored to the specified path.
3. If not found (cache miss), the job continues without restoring anything.
4. At the end of the job (if it succeeds), the contents of the specified path are saved under the key for future runs.

### Cache Keys and Lockfiles

Cache keys are typically generated by hashing a dependency lockfile. The lockfile (e.g., `package-lock.json`, `requirements.txt`, `Pipfile.lock`, `Gemfile.lock`) records the exact versions of every dependency. When the lockfile does not change, the hash does not change, and the cache is reused. When a dependency is added, updated, or removed, the lockfile changes, the hash changes, the cache misses, and fresh dependencies are installed and saved as a new cache.

```yaml
steps:
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}

  - run: npm ci
```

The `runner.os` prefix ensures Linux, Windows, and macOS caches are stored separately, because compiled packages are not cross-platform.

### Restore Keys (Fallback Cache)

If the exact key does not match, GitHub can fall back to a partial match using `restore-keys`. This is useful when the lockfile has changed but you still want to restore a nearby cache rather than starting from nothing.

```yaml
steps:
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-npm-
```

GitHub tries the keys in order. The first match is used. If the exact key fails, it tries to find any cache starting with `ubuntu-latest-npm-`. This restores an older cache, and the install step downloads only the changed packages. The new complete cache is then saved under the exact key.

### Cache Hit Detection

The `cache` action sets an output named `cache-hit`. You can use this to skip the install step entirely when the cache is restored exactly.

```yaml
steps:
  - id: cache-node
    uses: actions/cache@v4
    with:
      path: node_modules
      key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

  - name: Install dependencies
    if: steps.cache-node.outputs.cache-hit != 'true'
    run: npm ci
```

When the cache hits, `npm ci` is skipped entirely. When it misses, `npm ci` runs and the result is cached for the next run.

### Cache Paths by Language

| Language / Tool | Path to Cache | Key File |
|---|---|---|
| Node.js (npm) | `~/.npm` | `**/package-lock.json` |
| Node.js (yarn) | `~/.yarn/cache` | `**/yarn.lock` |
| Python (pip) | `~/.cache/pip` | `**/requirements.txt` |
| Python (pipenv) | `~/.local/share/virtualenvs` | `**/Pipfile.lock` |
| Java (Maven) | `~/.m2/repository` | `**/pom.xml` |
| Java (Gradle) | `~/.gradle/caches` | `**/*.gradle*` |
| Ruby (bundler) | `vendor/bundle` | `**/Gemfile.lock` |
| Go | `~/go/pkg/mod` | `**/go.sum` |
| Rust (Cargo) | `~/.cargo` | `**/Cargo.lock` |

### Cache Limits and Eviction

- The total cache storage per repository is 10 GB.
- Caches not accessed in 7 days are automatically deleted.
- When the 10 GB limit is reached, GitHub evicts the least recently used caches to make room.
- Caches created on a branch are accessible from that branch and from the default branch. Caches from the default branch are accessible to all branches.

### Using setup-* Actions for Automatic Caching

Many official setup actions include built-in caching via their `cache` input. This is simpler than manually configuring `actions/cache`.

```yaml
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: 20
      cache: 'npm'        # Automatically caches ~/.npm keyed on package-lock.json

  - uses: actions/setup-python@v5
    with:
      python-version: '3.12'
      cache: 'pip'        # Automatically caches pip packages

  - uses: actions/setup-java@v4
    with:
      distribution: 'temurin'
      java-version: '21'
      cache: 'maven'      # Automatically caches ~/.m2
```

When built-in caching is available in the setup action, prefer it over manual `actions/cache` configuration. It handles the cache path and key generation correctly for the ecosystem.
