# Chapter 6: Contexts, Expressions, and Built-in Functions

Expressions allow workflows to programmatically evaluate variables, contexts, and status checks prior to step execution. Expressions are encapsulated within `${{ }}` syntax.

## Contexts
Contexts are objects containing information regarding the workflow run, runner environment, jobs, and steps.

### Common Contexts
- `github`: Contains payload data related to the workflow run and trigger event (e.g., `github.actor`, `github.ref`).
- `runner`: Contains data about the runner executing the current job.
- `job`: Contains the status of the current job.
- `steps`: Contains data regarding previously executed steps within the job.
- `secrets` / `vars`: Provides access to defined repository secrets and variables.
- `inputs`: Provides access to parameters supplied via `workflow_dispatch`.

*(Reference: `07-test-context.yaml`)*

## Built-in Functions
GitHub provides built-in functions for use within expressions to perform string evaluation, JSON parsing, and hashing.

### `toJson()`
The `toJson()` function formats a context object into a readable JSON string. It is primarily utilized for debugging workflow configurations.
```yaml
steps:
  - name: Print GitHub Context
    run: echo "${{ toJson(github) }}"
```

### Logical and String Functions
Expressions support native string evaluation functions:
- `contains(searchString, searchValue)`: Returns true if the search string contains the specified value.
```yaml
if: contains(github.event.head_commit.message, 'HOTFIX')
```
- `startsWith(searchString, searchValue)` / `endsWith(searchString, searchValue)`: Evaluates string boundaries.
```yaml
if: startsWith(github.ref, 'refs/heads/feature/')
```

### `hashFiles()`
Generates a cryptographic hash from a specified file or directory path. This function is predominantly used to generate cache keys based on dependency lockfiles.
```yaml
key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

### `fromJSON()`
Parses a JSON string back into an object or array. This is required when passing complex data structures via string variables.

## Syntax Contexts: `${{ }}` vs `$VAR`
Determining whether to use expression syntax or native bash variables depends on the evaluation context.

- **Expressions (`${{ }}`)**: Must be used for `if:` conditionals, action inputs (`with:`), and workflow definitions. The expression is interpolated by GitHub before the runner executes the script.
- **Native Bash (`$VAR`)**: Recommended inside `run:` steps. Injecting `${{ }}` directly into a shell script can introduce script injection vulnerabilities if the variable contains unescaped characters or malicious input.

**Example Comparison:**
```yaml
env:
  GREETING: "Hello"

steps:
  # Not Recommended: Subject to script injection
  - run: echo "${{ env.GREETING }} World"
  
  # Recommended: Evaluated natively by the runner's shell
  - run: echo "$GREETING World"
```
