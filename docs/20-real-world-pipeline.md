# Chapter 20: Real-World CI/CD Pipeline

This chapter builds a complete CI/CD pipeline for a Node.js web application from start to finish. It integrates the concepts from every preceding chapter into a single working system. Read through this chapter after reading the earlier chapters. Every technique used here has a dedicated chapter that covers it in depth.

## The Application

The application is a Node.js REST API with the following structure:

```
my-api/
  src/
    app.js
    routes/
  tests/
    unit/
    integration/
  Dockerfile
  package.json
  package-lock.json
  .github/
    workflows/
      ci.yml
      deploy.yml
```

The pipeline will:
1. Run on every pull request: lint, test, build, and report results.
2. Run on every push to `main`: build a Docker image, push to the container registry, and deploy to staging automatically.
3. Deploy to production only after manual approval in the GitHub UI.

## Step 1: Understanding the Pipeline Structure

A CI/CD pipeline for this application has two logical phases:

**CI Phase (pull_request):** Validate that the code is correct before merging. This runs on every pull request and must pass before merging is allowed.

**CD Phase (push to main):** After code is merged to main, build the final artifact and deploy it. Production deployment requires human approval.

These two phases are defined in separate workflow files because they have different triggers and different requirements.

## Step 2: The CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

# Cancel any in-progress CI run for the same PR when a new commit is pushed
concurrency:
  group: ci-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write     # Needed to post test summary as a PR comment

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run linter
        run: npm run lint

  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run unit tests
        run: npm test -- --coverage --reporter json --outputFile=test-results.json
        continue-on-error: true    # Upload report even when tests fail
        id: unit-tests

      - name: Upload test results
        uses: actions/upload-artifact@v4
        with:
          name: unit-test-results
          path: |
            test-results.json
            coverage/
          retention-days: 7

      - name: Generate test summary
        run: |
          if [[ -f test-results.json ]]; then
            PASSED=$(jq '.numPassedTests' test-results.json)
            FAILED=$(jq '.numFailedTests' test-results.json)
            COVERAGE=$(jq '.coverageMap | to_entries | map(.value.s | to_entries | map(.value)) | flatten | (map(select(. == 1)) | length) / length * 100' test-results.json 2>/dev/null || echo "N/A")
            
            echo "## Unit Test Results" >> $GITHUB_STEP_SUMMARY
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "| Result | Count |" >> $GITHUB_STEP_SUMMARY
            echo "| --- | --- |" >> $GITHUB_STEP_SUMMARY
            echo "| Passed | $PASSED |" >> $GITHUB_STEP_SUMMARY
            echo "| Failed | $FAILED |" >> $GITHUB_STEP_SUMMARY
          fi

      - name: Fail job if tests failed
        if: steps.unit-tests.outcome == 'failure'
        run: exit 1

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    timeout-minutes: 20

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://testuser:testpassword@localhost:5432/testdb
          NODE_ENV: test
        run: npm run test:integration

  build-check:
    name: Build Check
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - name: Build Docker image
        run: docker build -t myapi:pr-${{ github.event.pull_request.number }} .

      - name: Verify image starts
        run: |
          docker run --rm -d -p 3000:3000 --name test-container \
            myapi:pr-${{ github.event.pull_request.number }}
          sleep 5
          curl --fail http://localhost:3000/health
          docker stop test-container
```

### What Each Job Does and Why

**lint:** Catches style and syntax errors before the more expensive tests run. Runs in parallel with other jobs because it has no dependencies.

**unit-tests:** Runs fast, isolated tests with no external dependencies. The `continue-on-error: true` pattern allows the test report to be uploaded even when tests fail, then explicitly fails the job afterward. This gives you the report regardless of outcome.

**integration-tests:** Starts a real PostgreSQL database using a service container. The tests run against it. This validates that the application works correctly with its actual database, not a mock.

**build-check:** Builds the Docker image and verifies the container starts and responds to a health check. Catches Dockerfile errors and broken startup sequences before they reach the deployment pipeline.

## Step 3: The Deploy Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write     # Needed to push to GitHub Container Registry
  id-token: write     # Needed for OIDC cloud authentication

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    name: Build and Push Image
    runs-on: ubuntu-latest
    timeout-minutes: 30
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.push.outputs.digest }}

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract image metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=,format=long
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push image
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    name: Deploy to Staging
    needs: build-and-push
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment:
      name: staging
      url: https://staging.myapi.example.com
    concurrency:
      group: deploy-staging
      cancel-in-progress: false    # Queue deployments, never cancel mid-deploy

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_STAGING_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Deploy to staging
        env:
          IMAGE_TAG: ${{ needs.build-and-push.outputs.image-tag }}
        run: |
          echo "Deploying image: $IMAGE_TAG"
          aws ecs update-service \
            --cluster staging-cluster \
            --service myapi \
            --force-new-deployment

      - name: Wait for deployment to complete
        run: |
          aws ecs wait services-stable \
            --cluster staging-cluster \
            --services myapi

      - name: Run smoke tests against staging
        run: |
          STAGING_URL="https://staging.myapi.example.com"
          curl --fail --retry 3 --retry-delay 5 "$STAGING_URL/health"
          echo "Smoke tests passed"

  deploy-production:
    name: Deploy to Production
    needs: [build-and-push, deploy-staging]
    runs-on: ubuntu-latest
    timeout-minutes: 30
    environment:
      name: production
      url: https://myapi.example.com
    concurrency:
      group: deploy-production
      cancel-in-progress: false

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_PROD_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Deploy to production
        env:
          IMAGE_TAG: ${{ needs.build-and-push.outputs.image-tag }}
        run: |
          echo "Deploying image: $IMAGE_TAG to production"
          aws ecs update-service \
            --cluster production-cluster \
            --service myapi \
            --force-new-deployment

      - name: Wait for production deployment
        run: |
          aws ecs wait services-stable \
            --cluster production-cluster \
            --services myapi

      - name: Verify production health
        run: curl --fail --retry 3 https://myapi.example.com/health

  notify-failure:
    name: Notify on Failure
    needs: [deploy-staging, deploy-production]
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Send failure notification
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: |
          curl -X POST "$SLACK_WEBHOOK" \
            -H 'Content-type: application/json' \
            --data "{
              \"text\": \"Deployment failed for commit ${{ github.sha }} by ${{ github.actor }}\",
              \"attachments\": [{
                \"color\": \"danger\",
                \"text\": \"Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"
              }]
            }"
```

## Step 4: GitHub Configuration Required

For this pipeline to work, configure the following in the repository settings:

**Container Registry access:** GitHub Container Registry (`ghcr.io`) is used. The `GITHUB_TOKEN` with `packages: write` permission handles authentication automatically.

**Environments:** Create `staging` and `production` environments in Settings. Configure the `production` environment with required reviewers. Add wait timer if desired. Restrict production to the `main` branch.

**Variables (Settings → Secrets and variables → Variables):**
- `AWS_STAGING_ROLE_ARN`: ARN of the IAM role to assume for staging deployments
- `AWS_PROD_ROLE_ARN`: ARN of the IAM role to assume for production deployments
- `AWS_REGION`: AWS region (e.g., `us-east-1`)

**Secrets:**
- `SLACK_WEBHOOK_URL`: Slack incoming webhook URL for failure notifications

**OIDC Configuration in AWS:** Create IAM roles in AWS that trust GitHub's OIDC provider. The trust policy should restrict which repository and which environment can assume the role. The staging role should only be assumable when the environment is `staging`, and the production role only when the environment is `production`.

**Branch protection on `main`:** Require status checks to pass before merging. Add the `lint`, `unit-tests`, `integration-tests`, and `build-check` jobs as required checks.

## Step 5: The Full Flow in Practice

**When a developer opens a pull request:**
1. The CI workflow triggers.
2. Four jobs run in parallel: lint, unit-tests, integration-tests, build-check.
3. Results appear in the PR checks section.
4. If any job fails, the PR cannot be merged.
5. The test summary appears in the Actions tab.

**When the PR is merged to main:**
1. The Deploy workflow triggers.
2. `build-and-push` builds the Docker image and pushes it to `ghcr.io` tagged with the commit SHA.
3. `deploy-staging` deploys to staging automatically. No human approval needed.
4. After staging deploys successfully, smoke tests run against the staging URL.
5. `deploy-production` pauses and waits for a required reviewer to approve in the GitHub UI.
6. After approval, production is updated with the same image that was tested in staging.
7. If either deployment fails, `notify-failure` sends a Slack notification.

**If a developer pushes directly to main without a PR:**
The same deploy workflow triggers. The CI checks still run against the staging deployment, but the PR checks were not enforced. This is why branch protection rules that require the CI workflow to pass are important.

## Key Design Decisions Explained

**Why two workflow files?** CI and CD have different triggers, different permissions, and different audiences. Separating them makes each file easier to reason about and prevents CI configuration from accidentally gaining deployment permissions.

**Why SHA-pinned actions?** SHA pins are in the deploy workflow where the stakes are highest. A compromised action in a deployment workflow could exfiltrate production credentials or deploy malicious code.

**Why `concurrency: cancel-in-progress: false` for deployments?** Unlike CI runs (where you only care about the latest result), deployments must complete fully. Cancelling a deployment mid-execution can leave the system in a broken half-deployed state.

**Why OIDC instead of stored AWS credentials?** Stored AWS access keys have an indefinite lifetime. OIDC credentials expire when the workflow run ends. A leaked OIDC credential is useless after expiry.

**Why `notify-failure` uses `if: failure()` with `needs`?** This job runs only when at least one upstream job has failed. The `failure()` function at the job level evaluates the `needs` array. Without the explicit `if: failure()`, the job would be skipped when upstream jobs fail because the default condition for jobs with `needs` is `success()`.
