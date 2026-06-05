# Chapter 0: YAML Primer for GitHub Actions

Every GitHub Actions workflow is written in YAML. Before writing a single workflow, understanding how YAML works saves hours of debugging indentation errors and unexpected type conversions. This chapter covers everything about YAML that is relevant to GitHub Actions, nothing more.

## What YAML Is

YAML is a text format for representing structured data. It is designed to be readable by humans. GitHub Actions uses YAML because workflow files need to be committed to the repository alongside code, and a plain-text format makes code review possible.

YAML files use the `.yml` or `.yaml` extension. Both extensions work in GitHub Actions.

## The One Rule That Matters Most: Indentation

YAML uses indentation to represent structure. Unlike Python, the number of spaces is flexible, but it must be consistent within a file. GitHub Actions convention is **2 spaces per indentation level**. Tabs are not allowed in YAML and will cause a parse error.

```yaml
# Correct: 2 spaces per level
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Hello
        run: echo "Hello"

# Wrong: mixed indentation causes parse errors
jobs:
  build:
      runs-on: ubuntu-latest   # 6 spaces instead of 4
```

Every time you go one level deeper in structure, add 2 more spaces.

## Key-Value Pairs (Mappings)

A key-value pair is written as `key: value`. The colon and space between them are both required.

```yaml
name: My Workflow
runs-on: ubuntu-latest
```

Values can be strings, numbers, booleans, or nested structures.

## Strings

Most string values in YAML do not need quotes. However, quotes are required in specific situations.

```yaml
# No quotes needed for simple strings
name: Build and Test
run: echo Hello

# Quotes required when the value contains a colon
name: "Step: Install dependencies"

# Quotes required when the value starts with a special character
name: "- this would be a list item without quotes"

# Quotes required when you want to preserve leading/trailing whitespace
value: "  padded  "
```

Both single quotes and double quotes work. Inside single quotes, no escape sequences are processed. Inside double quotes, `\n` means newline, `\\` means a literal backslash.

```yaml
# Double quotes: \n is a newline character
message: "Line 1\nLine 2"

# Single quotes: \n is literally the characters backslash and n
message: 'Line 1\nLine 2'
```

In practice, most GitHub Actions values use double quotes or no quotes.

## Booleans

YAML recognizes `true` and `false` as boolean values (without quotes). It also recognizes `yes`, `no`, `on`, `off` as booleans in older YAML versions. To avoid confusion, always quote these words when you mean them as strings.

```yaml
# These are boolean true
required: true
fail-fast: false

# Quote when you want the string "true", not the boolean
default: "true"
```

## Numbers

Unquoted numbers are interpreted as integers or floats. This becomes important in matrix strategies.

```yaml
# This is the integer 3, not the string "3"
timeout-minutes: 30

# In a matrix, 3.8 becomes the float 3.8, which GitHub renders as "3"
matrix:
  python-version: [3.8, 3.9]    # Wrong: 3.8 becomes "3"
  python-version: ['3.8', '3.9'] # Correct: treated as strings
```

## Lists (Sequences)

A list is written using a dash and a space before each item. The dash must be indented at the same level as the key it belongs to.

```yaml
# Block sequence (one item per line)
branches:
  - main
  - develop
  - 'feature/**'

# Inline sequence (all items on one line, using square brackets)
branches: [main, develop]
```

Both forms produce the same result. Block sequences are more readable for long lists. Inline sequences are common for short lists.

A list of mappings is written by putting the dash before the first key of each mapping:

```yaml
steps:
  - name: Step One          # First mapping in the list
    run: echo "one"

  - name: Step Two          # Second mapping in the list
    run: echo "two"
    id: step2
```

## Nested Mappings

A value can itself be a mapping. This creates nested structure.

```yaml
on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
  pull_request:
    branches:
      - main
```

Here `on` contains a mapping. That mapping contains `push` and `pull_request`. Each of those contains their own nested mappings.

## Multiline Strings

Two special characters control how multiline strings are handled.

### The Literal Block Scalar (`|`)
Preserves newlines exactly. Each line in the YAML block becomes a new line in the resulting string.

```yaml
steps:
  - run: |
      echo "Line one"
      echo "Line two"
      echo "Line three"
```

The shell receives three separate `echo` commands, each on its own line. This is the standard way to write multi-command `run` blocks.

### The Folded Block Scalar (`>`)
Replaces newlines with spaces. The entire block becomes one long line. Used for long strings that need to wrap for readability.

```yaml
options: >-
  --health-cmd pg_isready
  --health-interval 10s
  --health-timeout 5s
  --health-retries 5
```

The `>-` variant strips the trailing newline. The result is a single string:
`--health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5`

## Comments

Comments start with `#` and extend to the end of the line. Inline comments are supported.

```yaml
jobs:
  build:
    # This comment explains why this runner is used
    runs-on: ubuntu-latest
    timeout-minutes: 30  # Fail the job if it runs longer than 30 minutes
```

Comments are ignored by the YAML parser entirely. Use them to explain non-obvious configuration choices.

## Null Values

A key with no value, or the explicit value `null`, represents an absent value.

```yaml
# These are equivalent
on:
  push:

on:
  push: null
```

In GitHub Actions, `on: push:` with no further configuration means trigger on push to any branch. This is a common pattern.

## Expressions Inside YAML

GitHub Actions extends YAML with its own expression syntax: `${{ }}`. These expressions are evaluated by GitHub before the YAML is sent to the runner. They can appear inside string values.

```yaml
name: Deploy ${{ github.ref_name }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Actor is ${{ github.actor }}"
```

The entire `${{ }}` block is a string from YAML's perspective. The GitHub Actions engine processes these expressions after parsing the YAML.

## Common YAML Mistakes in GitHub Actions

**Mistake 1: Using a tab instead of spaces**
```yaml
jobs:
	build:     # Tab character before 'build' - parse error
```

**Mistake 2: Missing space after colon**
```yaml
name:Deploy  # Wrong: no space after colon
name: Deploy # Correct
```

**Mistake 3: Wrong indentation level for a list item**
```yaml
steps:
- name: Step One   # Wrong: dash is at column 0, but steps is at column 0 too
steps:
  - name: Step One # Correct: dash is indented under steps
```

**Mistake 4: Unquoted string that looks like a number**
```yaml
matrix:
  python-version: [3.10]   # 3.10 is the float 3.1 in YAML
  python-version: ['3.10'] # '3.10' is the string "3.10"
```

**Mistake 5: Forgetting the pipe for multiline `run` blocks**
```yaml
# Wrong: YAML treats this as a single broken string
steps:
  - run:
    echo "first"
    echo "second"

# Correct
steps:
  - run: |
      echo "first"
      echo "second"
```
