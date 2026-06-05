# Chapter 8: Matrix Strategies

A matrix strategy allows workflows to execute a job across multiple variable combinations simultaneously. It prevents redundancy when testing code against diverse environments or language versions.

## Defining a Matrix
A matrix evaluates arrays of defined variables and instantiates parallel job runs for every combination.

### Basic Matrix Definition
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [14, 16, 18]
    
    steps:
      - name: Environment Details
        run: echo "Executing on Node ${{ matrix.node-version }}"
```
This configuration generates three discrete job executions in parallel.

### Multi-Dimensional Matrix
Multiple arrays can be defined to create combinations.
```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        node-version: [14, 16, 18]
        os: [ubuntu-latest, windows-latest]
```
This configuration generates six parallel job executions (3 node versions × 2 operating systems).

### Matrix Value Type Gotcha
GitHub Actions evaluates YAML values natively. A value like `3.8` in a YAML file is interpreted as the floating-point number `3.8`, which GitHub then renders as `3` (truncating the decimal) when it is interpolated into a string context.

To preserve the intended string representation, always quote decimal values in a matrix definition.

```yaml
strategy:
  matrix:
    # Incorrect: 3.8 becomes "3", 3.10 becomes "3.1"
    python-version: [3.8, 3.9, 3.10]

    # Correct: values are treated as strings and rendered exactly
    python-version: ['3.8', '3.9', '3.10']
```
*(Reference: `12-strategy-matrix.yaml`)*

## Matrix Modifiers (`exclude` and `include`)
The matrix strategy supports inclusion and exclusion directives to refine job generation.

### `exclude` - Removing an Entire Axis Value
When an `exclude` entry specifies only one variable, GitHub removes every combination that contains that value across all dimensions.

```yaml
strategy:
  matrix:
    python-version: ['3.8', '3.9', '3.10']
    node-version: ['1.0', '2.0']
    exclude:
      - python-version: '3.9'   # Removes all combinations where python is 3.9
      - node-version: '2.0'     # Removes all combinations where node is 2.0
```

**Combinations generated** (from 6 total):
- `3.8 / 1.0` ✓
- ~~`3.8 / 2.0`~~ - excluded (node `2.0` rule)
- ~~`3.9 / 1.0`~~ - excluded (python `3.9` rule)
- ~~`3.9 / 2.0`~~ - excluded (both rules)
- `3.10 / 1.0` ✓
- ~~`3.10 / 2.0`~~ - excluded (node `2.0` rule)

### `exclude` - Removing a Specific Combination
When an `exclude` entry specifies multiple variables, only the exact combination matching all specified values is removed.

```yaml
strategy:
  matrix:
    python-version: ['3.8', '3.9', '3.10']
    node-version: ['1.0', '2.0']
    exclude:
      - python-version: '3.9'
        node-version: '2.0'   # Only this specific pair is removed
```

**Combinations generated** (from 6 total):
- `3.8 / 1.0` ✓
- `3.8 / 2.0` ✓
- `3.9 / 1.0` ✓
- ~~`3.9 / 2.0`~~ - excluded (exact match)
- `3.10 / 1.0` ✓
- `3.10 / 2.0` ✓

*(Reference: `12-strategy-matrix.yaml`)*

### `include`
New combinations or additional variables can be added to an existing matrix using `include`. Each `include` entry is appended as an additional job.
```yaml
strategy:
  matrix:
    node-version: [14, 16]
    os: [ubuntu-latest]
    include:
      - os: macos-11
        node-version: 12
```

## Matrix Concurrency and Error Handling

### `fail-fast`
By default, the `fail-fast` parameter is set to `true`. If any individual job within the matrix fails, all other in-progress jobs associated with the matrix are immediately cancelled.

**Example with a controlled failure:**
The following matrix generates 6 combinations. The `3.9 / 2.0` combination is configured to exit with an error.

```yaml
strategy:
  fail-fast: true
  matrix:
    python-version: ['3.8', '3.9', '3.10']
    node-version: ['1.0', '2.0']

steps:
  - run: |
      if [[ ${{ matrix.python-version }} == '3.9' && ${{ matrix.node-version }} == '2.0' ]]; then
        exit 1
      fi
      echo "Running ${{ matrix.python-version }} / ${{ matrix.node-version }}"
```

**Execution order and outcome with `fail-fast: true`:**

| Combination | Result |
|---|---|
| `3.8 / 1.0` | ✓ success |
| `3.8 / 2.0` | ✓ success |
| `3.9 / 1.0` | ✓ success |
| `3.9 / 2.0` | ✗ **fails** → triggers cancellation |
| `3.10 / 1.0` | ⊘ skipped (cancelled) |
| `3.10 / 2.0` | ⊘ skipped (cancelled) |

To allow the matrix to complete despite failures, set `fail-fast: false`.
*(Reference: `12-strategy-matrix.yaml`)*

### `max-parallel`
The maximum number of concurrent jobs initiated by a matrix can be restricted to avoid API rate limits or resource exhaustion.

```yaml
strategy:
  max-parallel: 5
  matrix: ...
```
