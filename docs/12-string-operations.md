# Chapter 12: String Operations

String manipulation is a constant requirement in CI/CD pipelines. Branch names need sanitizing before they can be used as Docker image tags. Commit SHAs need shortening for human-readable identifiers. Release versions need their `v` prefix stripped before being passed to a build tool. Environment names need uppercasing to match environment variable conventions.

GitHub Actions provides two layers for string work: a small set of built-in expression functions, and the full Bash parameter expansion syntax available inside `run:` blocks.

## When to Use Which Layer

**Use expression functions** when the operation happens outside a `run:` block: in `if:` conditions, in `name:` fields, or in `env:` values at the job or workflow level.

**Use Bash parameter expansion** when the operation is inside a `run:` block. Bash has a much richer set of string manipulation tools and is always available on Linux runners.

## Part 1: Native Expression Functions

These functions are evaluated by the GitHub Actions engine before the runner executes anything. They are available in any expression context.

### `contains(haystack, needle)`

Returns `true` if the haystack contains the needle string. Case-insensitive.

```yaml
# Check if commit message contains a keyword
if: contains(github.event.head_commit.message, '[skip ci]')

# Check if the branch name contains 'feature'
if: contains(github.ref_name, 'feature')
```

### `startsWith(string, prefix)` and `endsWith(string, suffix)`

```yaml
# Only run for tag pushes (refs/tags/...)
if: startsWith(github.ref, 'refs/tags/')

# Only run for release/* branches
if: startsWith(github.ref_name, 'release/')

# Skip backend-specific repositories
if: '!endsWith(github.repository, "-frontend")'
```

### `format(template, arg0, arg1, ...)`

Replaces `{0}`, `{1}`, etc. in a template. Useful for building strings from multiple context values without string concatenation hacks.

```yaml
env:
  IMAGE_NAME: ${{ format('{0}/{1}:{2}', 'ghcr.io', github.repository, github.ref_name) }}
  # Result: ghcr.io/myorg/myapp:main

  SLACK_MSG: ${{ format('Deployed {0} to {1} by {2}', github.ref_name, inputs.environment, github.actor) }}
  # Result: Deployed v1.2.3 to production by octocat
```

### `join(array, separator)`

Concatenates elements of an array into a single string with a separator.

```yaml
env:
  COMMIT_IDS: ${{ join(github.event.commits.*.id, ', ') }}
  # Result: abc1234, def5678, ghi9012
```

### `replace(string, search, replacement)`

Replaces all occurrences of a substring. Available as a built-in function in expressions.

```yaml
env:
  SAFE_REF: ${{ replace(github.ref_name, '/', '-') }}
  # Input:  feature/login-page
  # Result: feature-login-page
```

## Part 2: Shell Parameter Expansion (Bash)

Bash parameter expansion operates on shell variables at execution time inside `run:` blocks. These operations do not call external tools; they are handled directly by Bash, making them fast and always available on Linux runners.

The general pattern is to assign the context value to a shell variable first, then apply the expansion:

```yaml
steps:
  - name: String operation
    run: |
      INPUT="${{ github.ref_name }}"
      RESULT="${INPUT/some-operation}"
      echo "result=$RESULT" >> $GITHUB_OUTPUT
```

### Substring Extraction

Extracts a portion of a string by offset and length.

Syntax: `${variable:offset:length}`

```yaml
steps:
  - name: Create short SHA identifier
    run: |
      FULL_SHA="${{ github.sha }}"
      SHORT_SHA="${FULL_SHA:0:7}"          # First 7 characters
      echo "Short SHA: $SHORT_SHA"         # Output: a3f9b2c

  - name: Extract major version
    run: |
      VERSION="${{ github.ref_name }}"     # e.g. v1.23.456
      # Remove the 'v' prefix first, then take everything before the first dot
      STRIPPED="${VERSION#v}"              # 1.23.456
      MAJOR="${STRIPPED%%.*}"              # 1
      echo "Major version: $MAJOR"
```

### Search and Replace

Replace the first or all occurrences of a pattern within a variable.

Syntax:
- `${variable/search/replacement}` - first match only
- `${variable//search/replacement}` - all matches

```yaml
steps:
  - name: Sanitize branch name for Docker tag
    run: |
      BRANCH="${{ github.ref_name }}"      # e.g. feature/login-page
      SAFE="${BRANCH//\//-}"               # Replace all '/' with '-'
      echo "Docker tag: $SAFE"             # Output: feature-login-page

  - name: Replace spaces with underscores
    run: |
      INPUT="my app name"
      CLEAN="${INPUT// /_}"               # my_app_name
      echo "Clean name: $CLEAN"
```

### Case Conversion

Convert strings to uppercase or lowercase.

Syntax: `${variable^^}` for uppercase, `${variable,,}` for lowercase.

```yaml
steps:
  - name: Uppercase environment name
    run: |
      ENV="${{ inputs.environment }}"      # staging
      UPPER="${ENV^^}"                     # STAGING
      echo "Environment: $UPPER"

  - name: Lowercase image registry
    run: |
      REPO="${{ github.repository }}"      # MyOrg/MyApp
      LOWER="${REPO,,}"                    # myorg/myapp
      echo "Registry path: $LOWER"
```

Container registry paths must be lowercase. This pattern is used frequently when building Docker image names from the repository context.

### Default Values

Substitute a default when a variable is empty or unset.

Syntax: `${variable:-default_value}`

```yaml
steps:
  - name: Apply environment default
    run: |
      TARGET="${{ inputs.environment }}"
      # If the input was left blank, default to 'staging'
      FINAL="${TARGET:-staging}"
      echo "Deploying to: $FINAL"

  - name: Apply version default
    run: |
      VERSION="${{ inputs.version }}"
      BUILD_VERSION="${VERSION:-$(date +%Y%m%d%H%M%S)}"
      echo "Version: $BUILD_VERSION"
```

### Trimming Prefix and Suffix

Remove a specific pattern from the beginning or end of a string.

Syntax:
- `${variable#prefix}` - remove shortest prefix match
- `${variable##prefix}` - remove longest prefix match
- `${variable%suffix}` - remove shortest suffix match
- `${variable%%suffix}` - remove longest suffix match

```yaml
steps:
  - name: Strip 'v' prefix from version tag
    run: |
      TAG="${{ github.ref_name }}"         # v1.2.3
      VERSION="${TAG#v}"                   # 1.2.3
      echo "Pure version: $VERSION"

  - name: Get filename from path
    run: |
      FILE_PATH="src/components/Header.jsx"
      FILE_NAME="${FILE_PATH##*/}"         # Header.jsx  (longest prefix ending with /)
      DIR_NAME="${FILE_PATH%/*}"           # src/components  (shortest suffix starting with /)
      echo "File: $FILE_NAME"
      echo "Dir:  $DIR_NAME"

  - name: Strip file extension
    run: |
      FILENAME="deployment-config.yaml"
      BASENAME="${FILENAME%.*}"            # deployment-config
      EXTENSION="${FILENAME##*.}"          # yaml
      echo "Base: $BASENAME, Ext: $EXTENSION"
```

### String Length

Get the number of characters in a string.

Syntax: `${#variable}`

```yaml
steps:
  - name: Validate input length
    run: |
      INPUT="${{ inputs.name }}"
      LENGTH="${#INPUT}"
      if [[ $LENGTH -gt 64 ]]; then
        echo "Error: name must be 64 characters or fewer (got $LENGTH)"
        exit 1
      fi
```

### Combining Operations: A Real Pipeline Example

A common requirement is taking the current branch name and producing a valid Docker image tag.

```yaml
steps:
  - name: Generate image tag from branch
    id: tag
    run: |
      BRANCH="${{ github.ref_name }}"

      # Lowercase the entire string (registry requirement)
      LOWER="${BRANCH,,}"

      # Replace all forward slashes with hyphens
      CLEAN="${LOWER//\//-}"

      # Replace any remaining special characters with hyphens
      SAFE=$(echo "$CLEAN" | sed 's/[^a-z0-9-]/-/g')

      # Truncate to 128 characters (Docker tag limit)
      FINAL="${SAFE:0:128}"

      echo "Image tag: $FINAL"
      echo "tag=$FINAL" >> $GITHUB_OUTPUT

  - name: Build Docker image
    run: docker build -t "myapp:${{ steps.tag.outputs.tag }}" .
```
