# Chapter 15: Permissions and the GITHUB_TOKEN

## The `GITHUB_TOKEN`

Every workflow run is automatically provisioned with a `GITHUB_TOKEN` secret. GitHub generates this token at the start of each run with a set of permissions scoped to the repository that owns the workflow. The token expires when the run completes or after 24 hours, whichever comes first. No configuration is required to make it available.

The token is accessed via `${{ secrets.GITHUB_TOKEN }}`.

```yaml
steps:
  - name: Create Release
    uses: actions/create-release@v1
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### What `GITHUB_TOKEN` Can Do
The token authenticates as a GitHub App installation, not as a user. It can perform actions against the GitHub API within the scope of its permissions: creating releases, posting comments on issues and pull requests, pushing commits to the repository, managing deployments, and downloading artifacts from private repositories.

### Default Permission Level
GitHub repositories have a configurable default permission level for `GITHUB_TOKEN`. This is set in **Settings → Actions → General → Workflow permissions** and applies to all runs in that repository.

- **Read-only** (recommended for security): The token can read repository contents but cannot push or write.
- **Read and write**: The token can push commits, create releases, and modify repository resources.

## The `permissions` Block

The `permissions` block allows fine-grained control over `GITHUB_TOKEN` permissions at the workflow level or individual job level. This is the recommended approach over setting a broad repository default.

### Available Permission Scopes

| Scope | Controls access to |
|---|---|
| `contents` | Repository files, releases, tags |
| `issues` | Issues and issue comments |
| `pull-requests` | Pull request reviews and comments |
| `packages` | GitHub Packages (publish/download) |
| `deployments` | Deployment environments |
| `statuses` | Commit status checks |
| `checks` | Check runs and check suites |
| `security-events` | Code scanning alerts |
| `actions` | Workflow runs and artifacts |
| `id-token` | OIDC token for cloud provider authentication |

Each scope accepts `read`, `write`, or `none`.

### Workflow-Level Permissions
Defined at the top level of the workflow file. Applies to all jobs unless overridden at the job level.

```yaml
permissions:
  contents: read
  issues: write

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/labeler@v4
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
```

### Job-Level Permissions
Defined within an individual job. Overrides the workflow-level setting for that job only.

```yaml
permissions:
  contents: read   # Workflow default: read-only

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build

  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write   # Override: this job needs to create a release
    steps:
      - name: Publish Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Restricting All Permissions
Setting `permissions: {}` at the workflow level grants no permissions to `GITHUB_TOKEN`, effectively preventing it from interacting with the GitHub API. Individual jobs must then declare only the scopes they require.

```yaml
permissions: {}

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      deployments: write
      contents: read
    steps:
      - run: ./deploy.sh
```

## OIDC: Keyless Cloud Authentication

### The Problem with Long-Lived Secrets
Storing cloud provider credentials (AWS access keys, GCP service account keys, Azure service principals) as GitHub Secrets creates long-lived credentials that must be rotated manually and represent a persistent security risk if exposed.

### What OIDC Solves
OpenID Connect (OIDC) allows a GitHub Actions workflow to obtain short-lived, automatically-rotating credentials directly from a cloud provider without storing any static credentials in GitHub Secrets. The cloud provider trusts GitHub's identity token and exchanges it for temporary access credentials.

### How It Works

1. The workflow requests a JWT identity token from GitHub's OIDC provider. This token contains claims about the run (repository, branch, environment, actor).
2. The workflow sends this token to the cloud provider.
3. The cloud provider validates the token against a pre-configured trust policy.
4. If valid, the cloud provider returns short-lived access credentials (e.g., AWS temporary credentials via STS).
5. The workflow uses these credentials for the duration of the run.

### Enabling OIDC
The `id-token: write` permission is required to request an OIDC token from GitHub.

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy-to-aws:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          # No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY required

      - name: Deploy to S3
        run: aws s3 sync ./dist s3://my-bucket/
```

### Cloud Provider Trust Configuration
Before OIDC can be used, the cloud provider must be configured to trust GitHub's identity tokens. This involves:
- Registering `token.actions.githubusercontent.com` as a trusted identity provider.
- Defining a trust policy (IAM role condition in AWS, Workload Identity in GCP) that restricts which repositories, branches, or environments can assume the role.

OIDC is the recommended authentication method for any workflow that interacts with cloud infrastructure, as it eliminates the need to store and rotate static credentials.
