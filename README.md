# Detect Changed Files Action

A GitHub Action that analyzes changed files in your repository and categorizes them based on configurable pattern matching rules. This is useful for conditional workflow execution, selective testing, or determining which parts of your codebase have been modified.

## Features

- Detects changed files between git references
- Categorizes changes using pattern matching rules defined in a configuration file
- Returns a JSON object with boolean flags for each configured group
- Works with pull requests, pushes, and custom git references
- Runs in an isolated Docker container for security and consistency

## Usage

### Basic Example

```yaml
name: CI
on: [pull_request, push]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for git diff

      - name: Detect changed files
        id: changes
        uses: borisfaure/detect-changed-files-action@v1
        with:
          config-file: '.github/changes.conf'

      - name: Print changes
        run: echo '${{ steps.changes.outputs.changes }}'
```

### Conditional Job Execution

```yaml
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.changes.outputs.changes }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Detect changed files
        id: changes
        uses: borisfaure/detect-changed-files-action@v1
        with:
          config-file: '.github/changes.conf'

  test-frontend:
    needs: detect-changes
    if: fromJSON(needs.detect-changes.outputs.frontend).frontend == true
    runs-on: ubuntu-latest
    steps:
      - name: Run frontend tests
        run: npm test

  test-backend:
    needs: detect-changes
    if: fromJSON(needs.detect-changes.outputs.frontend).backend == true
    runs-on: ubuntu-latest
    steps:
      - name: Run backend tests
        run: go test ./...
```

## Inputs

### `config-file` (required)

Path to the configuration file that defines pattern matching rules for categorizing changed files.

Example: `.github/changes.conf`

### `base-ref` (optional)

Base reference for git diff comparison. If not specified, the action will automatically determine the base reference:
- For pull requests: `github.event.pull_request.base.sha`
- For push events: `github.event.before`
- Fallback: `HEAD^`

## Outputs

### `changes`

A JSON object containing boolean values for each configured group. Each key corresponds to a group name defined in your configuration file, with a value of `true` if any files in that group were changed, or `false` otherwise.

Example output:
```json
{
  "frontend": true,
  "backend": false,
  "docs": true,
  "ci": false
}
```

## Configuration File Format

Create a configuration file (e.g., `.github/changes.conf`) that defines groups and their associated file patterns. The exact format depends on the `detect-changed-files` tool, but typically follows a pattern-matching syntax.

Example configuration:
```
[frontend]
src/frontend/**
*.js
*.jsx
*.tsx
*.css

[backend]
src/backend/**
*.go
*.py

[docs]
docs/**
*.md
README.md

[ci]
.github/workflows/**
Dockerfile
```

Refer to the [detect-changed-files](https://github.com/borisfaure/detect-changed-files) documentation for detailed configuration syntax.

## Requirements

- `actions/checkout@v4` with `fetch-depth: 0` to ensure git history is available
- Docker must be available in the runner (available by default in GitHub-hosted runners)

## How It Works

1. Determines the base git reference (from input, PR base, or push event)
2. Runs `git diff --name-only` to get the list of changed files
3. Passes the file list to a Docker container running the `detect-changed-files` tool
4. The tool matches files against patterns defined in your configuration file
5. Returns a JSON object indicating which groups have changes

## License

MIT License - see [LICENSE](LICENSE) file for details.
