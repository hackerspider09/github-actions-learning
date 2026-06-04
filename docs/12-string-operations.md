# Chapter 12: String Operations

String manipulation is a frequent requirement in CI/CD pipelines, whether formatting release tags, parsing branch names, or generating dynamic deployment URLs. 

String operations in GitHub Actions are typically handled in two distinct layers: **Native Expression Functions** and **Shell Parameter Expansion**.

## 1. Native Expression Functions

GitHub Actions provides a limited set of built-in string functions. These are evaluated by the GitHub Actions engine *before* the runner executes the step. They are primarily used in `if` conditionals or action inputs.

### `contains()`
Returns `true` if a string contains a specified substring.
```yaml
if: contains(github.ref_name, 'feature')
```

### `startsWith()` and `endsWith()`
Evaluates the boundaries of a string.
```yaml
if: startsWith(github.ref_name, 'release/')
if: endsWith(github.repository, '-backend')
```

### `format()`
Replaces placeholders within a template string. Useful for constructing dynamic strings from multiple context variables.
```yaml
env:
  PR_MESSAGE: ${{ format('Pull request opened by {0} in {1}', github.actor, github.repository) }}
```

### `join()`
Concatenates an array of strings using a specified delimiter.
```yaml
env:
  TAG_LIST: ${{ join(github.event.commits.*.id, ', ') }}
```

## 2. Shell Parameter Expansion (Bash)

Because native expression functions are limited, complex string operations (such as substitution or slicing) must be executed natively within the runner's shell. On Linux runners (`ubuntu-latest`), the default shell is Bash.

### Substring Extraction (Slicing)
Extracts a portion of a string.
*Syntax: `${variable:offset:length}`*
```yaml
steps:
  - name: Extract Short SHA
    run: |
      FULL_SHA="${{ github.sha }}"
      SHORT_SHA="${FULL_SHA:0:7}"
      echo "Short SHA: $SHORT_SHA"
```

### Search and Replace
Replaces substrings within a variable. This is frequently used to sanitize branch names for use as Docker image tags.
*Syntax: `${variable/search/replace}` (first match) or `${variable//search/replace}` (all matches)*
```yaml
steps:
  - name: Sanitize Branch Name
    run: |
      RAW_BRANCH="feature/login-page"
      
      # Replace the first '/' with '-'
      SAFE_BRANCH="${RAW_BRANCH/\//-}"
      echo "Safe Branch: $SAFE_BRANCH" # Output: feature-login-page
```

### Case Conversion
Converts a string to uppercase or lowercase.
*Syntax: `${variable^^}` (uppercase) or `${variable,,}` (lowercase)*
```yaml
steps:
  - name: Convert Case
    run: |
      ENV_NAME="Production"
      
      UPPER_ENV="${ENV_NAME^^}" # Output: PRODUCTION
      LOWER_ENV="${ENV_NAME,,}" # Output: production
```

### Default Values
Assigns a default value if the variable is unset or empty.
*Syntax: `${variable:-default}`*
```yaml
steps:
  - name: Set Default Environment
    run: |
      TARGET_ENV="${{ inputs.target_env }}"
      
      # If TARGET_ENV is empty, use 'staging'
      FINAL_ENV="${TARGET_ENV:-staging}"
      echo "Deploying to $FINAL_ENV"
```

### Path Parsing (Basename and Dirname)
Extracts the filename or directory path from a full path string.
*Syntax: `${variable##*/}` (basename) or `${variable%/*}` (dirname)*
```yaml
steps:
  - name: Parse File Path
    run: |
      FILE_PATH="src/components/Header.jsx"
      
      FILE_NAME="${FILE_PATH##*/}" # Output: Header.jsx
      DIR_NAME="${FILE_PATH%/*}"   # Output: src/components
```

### Trimming Prefix or Suffix
Removes a specific pattern from the beginning or end of a string.
*Syntax: `${variable#prefix}` (remove prefix) or `${variable%suffix}` (remove suffix)*
```yaml
steps:
  - name: Trim String Boundaries
    run: |
      TAG_NAME="v1.2.3"
      
      # Remove 'v' prefix
      VERSION="${TAG_NAME#v}" # Output: 1.2.3
```
