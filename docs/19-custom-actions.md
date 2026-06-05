# Chapter 19: Custom Actions

A custom action is a packaged, reusable unit of code that you write yourself. Where composite actions (Chapter 10) bundle existing steps, custom actions are self-contained programs. They can be written in JavaScript/TypeScript or packaged as Docker containers. Either form is invoked the same way as any marketplace action, using `uses:`.

## When to Write a Custom Action

Write a custom action when:
- The logic you need does not exist in the marketplace and is complex enough that a `run:` script in every workflow would be unmanageable.
- The action needs to be shared across repositories as a versioned, publishable package.
- The action needs to interact with the GitHub API in complex ways that require a full programming language.
- The action needs isolated dependencies, a specific runtime, or a specific base environment (Docker action).

## The `action.yml` Metadata File

Every action type requires an `action.yml` file at the root of the action's directory. This file defines the action's interface: its name, description, inputs, and outputs.

```yaml
name: 'My Custom Action'
description: 'A description of what this action does'

inputs:
  input-name:
    description: 'Description of this input'
    required: true
  optional-input:
    description: 'This input has a default value'
    required: false
    default: 'default-value'

outputs:
  output-name:
    description: 'Description of this output'

runs:
  using: '...'      # 'composite', 'docker', or 'node20'
```

## Part 1: Docker Container Actions

### What They Are

A Docker action builds and runs a Docker container on the runner. The action's code runs inside the container, isolated from the runner's environment. This is useful when the action requires specific system libraries, non-Node runtimes, or a reproducible environment regardless of the host OS.

Docker actions only run on Linux runners.

### Directory Structure

```
my-docker-action/
  action.yml
  Dockerfile
  entrypoint.sh
```

### The `action.yml` for a Docker Action

```yaml
name: 'Image Analyzer'
description: 'Analyzes a Docker image for security vulnerabilities'

inputs:
  image-name:
    description: 'Docker image to analyze'
    required: true
  severity-threshold:
    description: 'Minimum severity level to report'
    required: false
    default: 'HIGH'

outputs:
  vulnerability-count:
    description: 'Number of vulnerabilities found'

runs:
  using: 'docker'
  image: 'Dockerfile'    # Build from the local Dockerfile
  args:
    - ${{ inputs.image-name }}
    - ${{ inputs.severity-threshold }}
```

Alternatively, reference a pre-built image from a registry:

```yaml
runs:
  using: 'docker'
  image: 'docker://alpine:3.18'
```

### The Dockerfile

```dockerfile
FROM alpine:3.18

RUN apk add --no-cache curl jq bash

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

### The Entrypoint Script

The entrypoint script receives the `args` from `action.yml` as positional parameters. Outputs are written to `$GITHUB_OUTPUT`.

```bash
#!/bin/bash
set -e

IMAGE_NAME="$1"
SEVERITY_THRESHOLD="$2"

echo "Analyzing image: $IMAGE_NAME"
echo "Threshold: $SEVERITY_THRESHOLD"

# Perform analysis (simplified example)
VULN_COUNT=$(curl -s "https://api.example.com/scan?image=${IMAGE_NAME}" | jq '.count')

# Write output
echo "vulnerability-count=${VULN_COUNT}" >> "$GITHUB_OUTPUT"

# Exit with non-zero if vulnerabilities found above threshold
if [[ $VULN_COUNT -gt 0 ]]; then
  echo "Found ${VULN_COUNT} vulnerabilities at or above ${SEVERITY_THRESHOLD}"
  exit 1
fi
```

### Using a Docker Action

```yaml
steps:
  - name: Analyze image
    id: scan
    uses: ./my-docker-action    # Local path to the action directory
    with:
      image-name: myapp:latest
      severity-threshold: HIGH

  - name: Report
    run: echo "Vulnerabilities found: ${{ steps.scan.outputs.vulnerability-count }}"
```

## Part 2: JavaScript Actions

### What They Are

JavaScript (or TypeScript compiled to JavaScript) actions run directly on the runner using Node.js. They start faster than Docker actions because there is no container to build or start. They run on Linux, Windows, and macOS.

JavaScript actions are the standard choice for actions that need to interact with the GitHub API, process data, or perform actions that do not require system isolation.

### Directory Structure

```
my-js-action/
  action.yml
  index.js          (or dist/index.js for compiled TypeScript)
  package.json
  node_modules/     (must be committed or bundled)
```

**Important:** The `node_modules` directory must be committed to the repository (or the code must be compiled into a single bundled file using a tool like `@vercel/ncc`). GitHub does not run `npm install` when invoking the action.

### The `action.yml` for a JavaScript Action

```yaml
name: 'Version Bumper'
description: 'Reads package.json and outputs the current version'

inputs:
  package-path:
    description: 'Path to package.json'
    required: false
    default: 'package.json'

outputs:
  current-version:
    description: 'The version string from package.json'
  next-patch-version:
    description: 'The version with patch number incremented'

runs:
  using: 'node20'      # Specify the Node.js major version to use
  main: 'index.js'     # The entry point file
```

Node.js versions available: `node16`, `node20`. Use `node20` for new actions.

### The Action Code (`index.js`)

The `@actions/core` package provides the API for reading inputs, writing outputs, setting failure status, and logging.

```javascript
const core = require('@actions/core');
const fs = require('fs');
const path = require('path');

async function run() {
  try {
    // Read inputs declared in action.yml
    const packagePath = core.getInput('package-path');

    // Read the package.json file
    const absolutePath = path.resolve(process.env.GITHUB_WORKSPACE, packagePath);
    const packageJson = JSON.parse(fs.readFileSync(absolutePath, 'utf8'));
    const currentVersion = packageJson.version;

    // Calculate next patch version
    const parts = currentVersion.split('.');
    parts[2] = String(parseInt(parts[2], 10) + 1);
    const nextPatch = parts.join('.');

    // Write outputs declared in action.yml
    core.setOutput('current-version', currentVersion);
    core.setOutput('next-patch-version', nextPatch);

    // Log messages (visible in the step log)
    core.info(`Current version: ${currentVersion}`);
    core.info(`Next patch version: ${nextPatch}`);

  } catch (error) {
    // Mark the step as failed with an error message
    core.setFailed(error.message);
  }
}

run();
```

### The `@actions/core` API Reference

| Method | Description |
|---|---|
| `core.getInput('name')` | Read a declared input value |
| `core.getInput('name', { required: true })` | Read a required input; throws if missing |
| `core.getBooleanInput('name')` | Read an input as a boolean |
| `core.setOutput('name', value)` | Write an output value |
| `core.setFailed('message')` | Fail the step with a message |
| `core.info('message')` | Log an informational message |
| `core.warning('message')` | Log a warning (shown in UI) |
| `core.error('message')` | Log an error without failing the step |
| `core.setSecret('value')` | Register a value to be masked in all future log output |
| `core.exportVariable('name', value)` | Set an environment variable for subsequent steps (writes to `$GITHUB_ENV`) |
| `core.addPath('directory')` | Add a directory to PATH for subsequent steps (writes to `$GITHUB_PATH`) |
| `core.startGroup('name')` | Start a collapsible log group in the UI |
| `core.endGroup()` | End the current log group |
| `core.summary.addHeading('text')` | Add a heading to the step summary |
| `core.summary.addTable(data)` | Add a table to the step summary |
| `core.summary.write()` | Flush the summary to `$GITHUB_STEP_SUMMARY` |

### Building with TypeScript

TypeScript is the recommended language for complex actions. Compile it to JavaScript before publishing. The `@vercel/ncc` bundler compiles everything including `node_modules` into a single `dist/index.js` file, which means you only commit the compiled output rather than the entire `node_modules` directory.

```json
// package.json
{
  "scripts": {
    "build": "ncc build src/index.ts -o dist --source-map --license licenses.txt"
  },
  "devDependencies": {
    "@types/node": "^20",
    "@vercel/ncc": "^0.38",
    "typescript": "^5"
  },
  "dependencies": {
    "@actions/core": "^1.10",
    "@actions/github": "^6"
  }
}
```

```yaml
# action.yml points to the compiled file
runs:
  using: 'node20'
  main: 'dist/index.js'
```

The `@actions/github` package provides an authenticated GitHub API client pre-configured with the `GITHUB_TOKEN`. It wraps `@octokit/rest`.

```javascript
const github = require('@actions/github');

async function run() {
  const token = core.getInput('github-token');
  const octokit = github.getOctokit(token);
  const context = github.context;

  await octokit.rest.issues.createComment({
    owner: context.repo.owner,
    repo: context.repo.repo,
    issue_number: context.payload.pull_request.number,
    body: 'Automated comment from custom action'
  });
}
```

## Publishing an Action to the Marketplace

To publish an action publicly:
1. The `action.yml` must be at the root of the repository (not in a subdirectory).
2. The repository must be public.
3. Create a release in the repository. GitHub prompts you to publish it to the marketplace during release creation.

For internal organizational use, actions do not need to be published to the marketplace. They can be referenced by their repository path within any workflow in the organization.
