# Chapter 21: DevSecOps in GitHub Actions

## What DevSecOps Is

DevSecOps is integrating security into every stage of the DevOps pipeline so vulnerabilities are detected early during development instead of after deployment. Security checks are added at each CI/CD pipeline stage - Code, Build, Test, Deploy - rather than as a single audit at the end.

The traditional model runs security last. By the time a vulnerability is found, the code was written weeks ago, context is gone, and fixing it delays a release. DevSecOps follows a **shift left mindset**: do security after each step rather than at the end. Finding a problem in a pull request costs a developer minutes. Finding it in production costs days of incident response.

```
Developer Push
      |
      v
Secret Scanning        (Gitleaks, TruffleHog)
      |
      v
SAST (Static Code Analysis)   (CodeQL, SonarQube, Semgrep)
      |
      v
SCA (Dependency Scan)         (Trivy, OWASP Dependency-Check, Snyk)
      |
      v
Build Application             (Maven, Gradle, pip, npm)
Unit Tests                    (JUnit, pytest, Jest)
      |
      v
Dockerfile Lint               (Hadolint)
      |
      v
Build Container Image         (Docker)
      |
      v
Container Image Scan          (Trivy, Clair, Anchore)
      |
      v
Push Image to Registry        (ECR, Docker Hub)
      |
      v
Deploy                        (Docker Compose, Kubernetes)
      |
      v
Runtime Security / Monitor    (Prometheus, Falco)
```

## Types of Security Scanning

| Type | Full Name | What It Scans |
|---|---|---|
| SAST | Static Application Security Testing | Source code, without running the application |
| SCA | Software Composition Analysis | Third-party dependencies and libraries |
| DAST | Dynamic Application Security Testing | The running application |
| Container Scan | Image Security Scan | Docker image layers, OS packages |
| Secrets Scan | Credential Detection | Git history and source files for leaked credentials |
| Dockerfile Lint | Infrastructure Code Check | Dockerfile for security misconfigurations before building |

## SOC2 and Compliance

SOC2 (Service Organization Control 2) is a compliance framework that requires organizations to demonstrate controls around security, availability, and confidentiality. GitHub Actions helps satisfy SOC2 requirements by providing auditable, automated security gates on every code change. Evidence of automated scanning, required reviewers before production deployment, and immutable audit logs of every pipeline run can all be produced directly from GitHub Actions.

## Stage 1: Secrets Scanning

### What It Is

Automated detection of credentials, API keys, tokens, and other sensitive values committed to source code or Git history.

### Why It Exists

Developers occasionally commit secrets by mistake. Once committed to Git, a secret exists permanently in history. Deleting the file in a follow-up commit does not remove it from history - anyone who clones the repository can read all past commits.

### How It Works

Secrets scanners use regular expressions matched against known credential formats: AWS access keys (`AKIA...`), GitHub tokens (`ghp_...`), private key PEM headers, Stripe keys, Slack webhooks, and high-entropy strings that match no known pattern but are statistically likely to be secrets.

### Why `fetch-depth: 0` Is Required

By default `actions/checkout` fetches only the most recent commit (a shallow clone). A shallow clone hides all previous commits from the scanner. If a secret was committed three days ago and then partially obscured, it remains invisible without full history. `fetch-depth: 0` fetches the entire Git history so every commit is scanned.

Gitleaks also needs `GITHUB_TOKEN` - this token is created by GitHub for the runner and stored automatically as a secret on every run. No manual configuration is required.

```yaml
jobs:
  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # Full history required to scan all commits

      - name: Scan for secrets
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### What to Do When a Secret Is Found

1. Rotate the credential immediately. Assume it is compromised even if the repository is private.
2. Remove it from Git history using `git filter-repo` or BFG Repo Cleaner.
3. Force-push the cleaned history.
4. If the repository is public, notify the service provider.

### Tools

| Tool | Notes |
|---|---|
| Gitleaks | Open-source, 600+ built-in rules, configurable via `.gitleaks.toml` |
| TruffleHog | Open-source, detects high-entropy strings and known patterns |
| GitHub Secret Scanning | Built into GitHub for public repos and Advanced Security subscribers |
| detect-secrets | Yelp's tool, designed for pre-commit and CI integration |

## Stage 2: SAST - Static Application Security Testing

### What It Is

Analysis of source code without executing the application to find security vulnerabilities in the application's own logic.

### Why It Exists

Developers write code that inadvertently creates vulnerabilities: SQL injection, cross-site scripting, path traversal, command injection, insecure cryptography, hardcoded passwords. These patterns can be detected by analyzing code structure and data flow without running anything.

### How It Works Internally

1. **Parse source code into AST (Abstract Syntax Tree):** The source files are parsed into a tree structure that represents every function, variable, and expression.
2. **Apply security rules:** Rules are matched against the AST. A rule might flag any call to `exec()` with user input, or any string concatenation feeding into a SQL query.
3. **Data flow analysis:** The tool traces how data moves through the application - from input sources (HTTP request parameters, file reads, environment variables) to dangerous sinks (SQL queries, shell commands, file writes). If user-controlled data reaches a sink without passing through a sanitization function, that is a vulnerability.
4. **Report:** Results are produced with file, line number, vulnerability class, and severity.

### What SAST Detects

- SQL injection
- Cross-site scripting (XSS)
- Path traversal and directory traversal
- Command injection
- Insecure use of cryptographic functions
- Hardcoded credentials and passwords
- Unsafe functions
- Code quality issues and code smells
- Memory leaks (in compiled languages)

### GitHub Actions Implementation

```yaml
jobs:
  sast:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: java    # java, python, javascript, go, ruby, cpp, csharp, swift

      # For compiled languages (Java, C++, Go): CodeQL instruments the compiler.
      # It intercepts the build to trace data flow through compiled code.
      # For interpreted languages (Python, JavaScript): no build step is needed.
      - name: Build code
        run: mvn clean package -DskipTests

      - name: Run CodeQL analysis
        uses: github/codeql-action/analyze@v3
```

### Tools

| Tool | Languages | Notes |
|---|---|---|
| CodeQL | Java, Python, JS/TS, Go, Ruby, C/C++, C#, Swift | GitHub-native, free for public repos |
| SonarQube | 30+ languages | Self-hosted or SonarCloud, code quality + security |
| Semgrep | 30+ languages | Open-source, fast, large rule registry |
| Bandit | Python | Lightweight Python-specific scanner |
| ESLint (security plugins) | JavaScript/TypeScript | Integrates with existing linting |
| Brakeman | Ruby on Rails | Rails-specific, very fast |
| Gosec | Go | Go-specific security scanner |

## Stage 3: SCA - Software Composition Analysis

### What It Is

Automated detection of known security vulnerabilities in third-party libraries and dependencies the application uses.

### Why It Exists

Modern applications depend on hundreds of open-source libraries. A perfectly written application can be vulnerable because a library it depends on has a known CVE. A prime example is the famous **Log4j** vulnerability - millions of Java applications had no flaws in their own code, but imported the vulnerable Log4j library, instantly making them susceptible to remote attacks. SCA is different from SAST: SAST analyzes the application's own code, SCA analyzes what the application imports.

### How It Works Internally

1. **Read dependency declaration files:** The scanner reads `pom.xml`, `requirements.txt`, `package.json`, `go.mod`, `Gemfile.lock`, `Cargo.lock`, etc.
2. **Extract all dependencies and their versions.**
3. **Check against vulnerability databases:** CVE (National Vulnerability Database), NVD, GitHub Advisory Database, OSV, VulnDB.
4. **Version comparison:** The exact version in use is compared against the range of affected versions for each known CVE.
5. **Report:** Results list the library name, version, CVE identifier, CVSS severity, and whether a fixed version is available.

### GitHub Actions Implementation

```yaml
jobs:
  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Dependency scan
        uses: aquasecurity/trivy-action@0.35.0
        with:
          scan-type: 'fs'
          scan-ref: '.'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'
```

### Managing False Positives with `.trivyignore`

No scanner is perfect. Sometimes an SCA tool flags a vulnerability in a dependency, but your application never calls the vulnerable function. Or perhaps a patch is not yet available, and the business has accepted the risk. 

To prevent Trivy from failing the pipeline on these known, accepted risks, create a `.trivyignore` file in the root of the repository (`github.workspace`). Add the CVE IDs (one per line) that you wish to ignore:

```text
# .trivyignore
CVE-2023-12345
CVE-2023-67890
```
Trivy automatically detects this file and excludes these CVEs from the failure threshold.

### Tools

| Tool | Notes |
|---|---|
| Trivy | Open-source, fast, 20+ ecosystems, also does image and IaC scanning |
| OWASP Dependency-Check | Open-source, widely used for Java/Maven/Gradle |
| Snyk | Commercial with free tier, detailed remediation advice |
| pip-audit | Python-specific, queries PyPI Advisory Database |
| npm audit | Built into npm |
| Safety | Python-specific CVE checker |

## Stage 4: Dockerfile Linting (Hadolint)

### What It Is

Static analysis of the Dockerfile itself before the image is built, checking for security misconfigurations, bad practices, and CVE-prone patterns in the image construction instructions.

### Why It Exists

The Dockerfile defines what goes into the container. A Dockerfile that uses `latest` tags (unpinned base images), runs as `root`, installs unnecessary packages, or skips a `HEALTHCHECK` creates a container that is harder to secure and audit. These issues are fast and cheap to catch before the image is built.

### What Hadolint Checks

- Using `latest` tag on base images (unpinned, non-reproducible builds)
- Running the container process as root (no `USER` instruction)
- Using `apt-get upgrade` (leads to non-reproducible images)
- Combining `RUN apt-get update` and `apt-get install` in separate layers (cache invalidation issue)
- Not pinning package versions in `apt-get install`
- Missing `HEALTHCHECK` instruction
- Using `ADD` when `COPY` is sufficient
- Exposing unnecessary ports

### GitHub Actions Implementation

```yaml
jobs:
  dockerfile-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          failure-threshold: warning
```

## Stage 5: Container Image Scanning

### What It Is

Scanning a built Docker image for vulnerabilities in OS packages and application libraries after the image has been constructed.

### Why It Exists

SCA at the source level reads the application's dependency files. It cannot see the OS packages installed by the base image (glibc, openssl, curl, libssl, etc.) because those are not declared in `pom.xml` or `package.json`. They come from the `FROM` instruction in the Dockerfile. Container image scanning reads every layer of the image and checks all packages - OS-level and application-level - against CVE databases.

### How Trivy Works for Image Scanning

Trivy extracts the full list of installed packages from every image layer (the base image layer, every `RUN` layer, the `COPY` layer). It then queries its embedded vulnerability database, which aggregates data from NVD, GitHub Advisory Database, Red Hat, Alpine, Debian, and others. Results are classified by severity.

### Severity Levels

| Severity | CVSS Score | Meaning |
|---|---|---|
| Critical | 9.0 - 10.0 | Remotely exploitable, no authentication needed, high impact |
| High | 7.0 - 8.9 | Significant impact, some conditions required |
| Medium | 4.0 - 6.9 | Exploitable but requires specific conditions |
| Low | 0.1 - 3.9 | Low exploitability or low impact |
| Unknown | N/A | Pending CVSS assignment |

`CRITICAL` always blocks. `HIGH` usually blocks. `Medium` and `Low` are reported but typically do not block the pipeline.

### GitHub Actions Implementation

```yaml
jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Scan image
        uses: aquasecurity/trivy-action@0.35.0
        with:
          scan-type: 'image'
          image-ref: 'myapp:${{ github.sha }}'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL'

  push:
    needs: build-and-scan    # Push only if scan passed
    runs-on: ubuntu-latest
    steps:
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Push image
        run: docker push myapp:${{ github.sha }}
```

### Tools

| Tool | Notes |
|---|---|
| Trivy | Open-source, fast, single binary, recommended |
| Clair | Open-source from Quay, often embedded in private registries |
| Anchore/Grype | Open-source, high accuracy |
| Snyk Container | Commercial with free tier |
| Docker Scout | Built into Docker Hub |

## Stage 6: DAST - Dynamic Application Security Testing

### What It Is

Security testing that executes against a running instance of the application, sending HTTP requests and analyzing responses to find vulnerabilities.

### Why It Exists

SAST analyzes code without running it. It cannot find vulnerabilities that only appear at runtime: misconfigurations in the web server, vulnerabilities in how the framework processes certain request types, or flaws that emerge from the interaction of multiple components. DAST treats the application as a black box and probes it the same way an attacker would.

### How It Works

1. **Crawl the application:** The scanner discovers all endpoints, forms, and parameters by navigating the application like a browser.
2. **Send attack payloads:** Crafted inputs are sent to trigger known vulnerability classes - SQL injection strings, XSS payloads, path traversal sequences, command injection attempts.
3. **Analyze responses:** Responses are checked for signs that the payload succeeded (error messages revealing database structure, reflected scripts, unexpected file contents).
4. **Report:** Findings are classified by vulnerability type and severity.

### When It Runs

Against a running deployment - a review environment or staging environment. DAST cannot run against source code; it requires a live application. In GitHub Actions, a deploy-to-test-env job must run before the DAST job.

### GitHub Actions Implementation

```yaml
jobs:
  dast:
    runs-on: ubuntu-latest
    needs: deploy-test-environment
    steps:
      - name: Run OWASP ZAP scan
        uses: zaproxy/action-full-scan@v0.12.0
        with:
          target: 'https://staging.example.com'
          cmd_options: '-a'    # Active scan: sends attack payloads
```

### Tools

| Tool | Notes |
|---|---|
| OWASP ZAP | Open-source, full-featured, passive and active modes |
| Burp Suite | Commercial, widely used by security professionals |
| Nuclei | Open-source, template-based, fast |
| Nikto | Open-source, focused on server misconfiguration |
| StackHawk | Commercial, built for CI/CD integration |

## Stage 7: CD Deployment Pattern

The Continuous Deployment (CD) phase begins only after the CI pipeline has successfully built, scanned, and pushed the container image. 

A common and secure deployment pattern for a Docker-based application on a remote server involves three steps:

1. **Checkout code:** Retrieve the `docker-compose.yaml` file from the repository.
2. **Transfer files:** Use SCP (Secure Copy Protocol) to send the `docker-compose.yaml` file to the remote server directory.
3. **SSH and Deploy:** Execute commands remotely to log in to the registry and start the application.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Copy Compose File
        uses: appleboy/scp-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "docker-compose.yaml"
          target: "/home/ubuntu/app"

      - name: Deploy Application
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/ubuntu/app
            
            # Authenticate to registry securely via standard input
            echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "myuser" --password-stdin
            
            # Pull the latest scanned image and start the app
            docker compose down || true
            docker compose up -d --pull always
```

Notice the `echo "password" | docker login --password-stdin` pattern. Passing the password via standard input prevents the secret from appearing in the server's process list (`ps aux`) or bash history.

## The Golden Rule: One Tool for Multiple Scan Types

Many companies use one tool for multiple security categories rather than adopting a different tool per scan type. This reduces configuration overhead, standardizes output formats, and simplifies pipeline maintenance.

**Trivy is the most common example.** A single tool, multiple scan types:

```bash
trivy fs .                          # SCA: scan project filesystem for dependency CVEs
trivy image myapp:latest            # Container: scan built Docker image
trivy config ./terraform            # IaC: scan Terraform for misconfigurations
trivy repo https://github.com/...   # Remote: scan a public repository
trivy sbom --format spdx ./         # SBOM: generate software bill of materials
```

In GitHub Actions, you use the same `aquasecurity/trivy-action` for all of these, changing only the `scan-type` input.

Similarly, **Semgrep** covers SAST across 30+ languages and has community rules for dozens of vulnerability classes, reducing the need for language-specific tools.

## Language Ecosystem Tools Reference

| Category | Java | Python | Node.js |
|---|---|---|---|
| Build | Maven, Gradle | pip | npm, yarn |
| Test | JUnit | pytest | Jest, Mocha |
| SAST | CodeQL, SonarQube | Bandit, Semgrep | Semgrep, ESLint |
| SCA | OWASP Dependency-Check, Trivy | pip-audit, Safety, Trivy | npm audit, Snyk, Trivy |
| Image scan | Trivy | Trivy | Trivy |
| Dependency file | `pom.xml` | `requirements.txt` | `package.json`, `package-lock.json` |

## CI vs CD: What Runs Where

### CI (Continuous Integration) - runs on every pull request and push

- Code quality and linting
- SAST
- SCA (dependency scan)
- Secrets scanning (Gitleaks)
- Application build
- Unit tests
- Dockerfile lint (Hadolint)
- Container image build and scan

### CD (Continuous Deployment) - runs after CI passes on the main branch

- Push image to registry (ECR, Docker Hub)
- Deploy to server (Docker Compose, Kubernetes)
- DAST against staging environment
- Promote to production (with required reviewers if applicable)
- Runtime security monitoring (Prometheus, Falco)

## Full Pipeline Stage Order

```
Commit pushed
      |
      v
Secrets scan          (Gitleaks)           -- fast, blocks all downstream if credential found
      |
      +----------+
      |          |
      v          v
    SAST        SCA                        -- run in parallel, both must pass
      |          |
      +----------+
      |
      v
Build application     (Maven/pip/npm)
Unit tests            (JUnit/pytest/Jest)
      |
      v
Dockerfile lint       (Hadolint)
      |
      v
Build container image (Docker)
Container image scan  (Trivy)
      |
      v
Push image to registry (ECR / Docker Hub)  -- only if image scan passed
      |
      v
Deploy to test environment
DAST                  (OWASP ZAP)
      |
      v
Deploy to production  (manual approval gate via GitHub environment)
      |
      v
Runtime monitoring    (Prometheus, Falco)
```

## Uploading Results to GitHub Security Tab

Most tools can output results in SARIF format. Uploading SARIF makes findings visible in the repository's Security tab where they can be triaged, assigned, and tracked.

```yaml
      - name: Scan
        uses: aquasecurity/trivy-action@0.35.0
        with:
          format: 'sarif'
          output: 'results.sarif'

      - name: Upload to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()    # Upload even when the scan step failed
        with:
          sarif_file: 'results.sarif'
```

`if: always()` is required. Without it, a failed scan step (because vulnerabilities were found) causes the upload step to be skipped, making results invisible in the Security tab.

## Using Reusable Workflows for DevSecOps

Each security stage is best defined as a separate reusable workflow. A main orchestrator calls them in order. `secrets: inherit` passes repository secrets to child workflows that need them (image push, deployment). Scan-only workflows receive no secrets.

```yaml
# main.yml
jobs:
  secrets-scan:
    uses: ./.github/workflows/secrets-scan.yml

  sast:
    uses: ./.github/workflows/sast.yml

  sca:
    uses: ./.github/workflows/sca.yml

  build-and-scan:
    needs: [secrets-scan, sast, sca]
    uses: ./.github/workflows/build-scan.yml
    secrets: inherit    # Needs registry credentials

  deploy:
    needs: [build-and-scan]
    uses: ./.github/workflows/deploy.yml
    secrets: inherit    # Needs SSH key / cloud credentials
```

This structure means each security concern is isolated, independently testable, and reusable across multiple projects.
