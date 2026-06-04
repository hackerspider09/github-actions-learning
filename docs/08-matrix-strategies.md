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

## Matrix Modifiers (`exclude` and `include`)
The matrix strategy supports inclusion and exclusion directives to refine job generation.

### `exclude`
Specific variable combinations can be omitted from execution using `exclude`.
```yaml
strategy:
  matrix:
    node-version: [14, 16, 18]
    os: [ubuntu-latest, windows-latest]
    exclude:
      - os: windows-latest
        node-version: 14
```
*(Reference: `12-strategy-matrix.yaml`)*

### `include`
New combinations or specific variables can be added to an existing matrix using `include`.
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
By default, the `fail-fast` parameter is set to `true`. If any individual job within the matrix fails, all other in-progress jobs associated with the matrix are immediately cancelled. To allow the matrix to complete despite failures, set `fail-fast: false`.

```yaml
strategy:
  fail-fast: false
  matrix: ...
```

### `max-parallel`
The maximum number of concurrent jobs initiated by a matrix can be restricted to avoid API rate limits or resource exhaustion.

```yaml
strategy:
  max-parallel: 5
  matrix: ...
```
