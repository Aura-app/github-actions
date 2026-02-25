# Cursor Code Review Action

Automated AI code review using [Cursor](https://cursor.com), designed for monorepos. Runs scoped reviews in parallel with configurable per-folder guidelines.

## Quick Start

### 1. Add a scopes config file

Create `.cursor-review-scopes.json` in your repo root:

```json
{
  "scopes": [
    {
      "path": "backend/",
      "guidelines": "backend/REVIEW_GUIDELINES.md",
      "label": "Backend"
    },
    {
      "path": "frontend/",
      "guidelines": "frontend/REVIEW_GUIDELINES.md",
      "label": "Frontend"
    }
  ]
}
```

For a single-project repo (not a monorepo), use one scope covering everything:

```json
{
  "scopes": [
    {
      "path": "./",
      "guidelines": "REVIEW_GUIDELINES.md",
      "label": "All"
    }
  ]
}
```

### 2. Add review guidelines

Create a `REVIEW_GUIDELINES.md` at the repo root with your general review rules, and a `REVIEW_GUIDELINES.md` inside each scope folder with folder-specific rules.

### 3. Add the caller workflow

Copy the following into `.github/workflows/cursor-code-review.yml` in your repo (also available in the [examples](examples/) directory):

```yaml
name: Cursor Code Review

on:
  pull_request:
    branches: ["*"]
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  detect-changes:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.build-matrix.outputs.matrix }}
      has_changes: ${{ steps.build-matrix.outputs.has_changes }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Build review matrix from config
        id: build-matrix
        shell: bash
        run: |
          set -euo pipefail
          git fetch origin "${{ github.base_ref }}"
          changed="$(git diff --name-only "origin/${{ github.base_ref }}...HEAD")"
          entries=()
          while IFS= read -r scope; do
            p=$(echo "$scope" | jq -r '.path')
            if echo "$changed" | grep -qE "^${p}"; then
              entries+=("$scope")
            fi
          done < <(jq -c '.scopes[]' .cursor-review-scopes.json)
          if [ ${#entries[@]} -eq 0 ]; then
            echo "has_changes=false" >> "$GITHUB_OUTPUT"
            echo 'matrix={"include":[]}' >> "$GITHUB_OUTPUT"
          else
            matrix=$(printf '%s\n' "${entries[@]}" | jq -sc '{include:.}')
            echo "has_changes=true" >> "$GITHUB_OUTPUT"
            echo "matrix=$matrix" >> "$GITHUB_OUTPUT"
          fi

  review:
    needs: detect-changes
    if: needs.detect-changes.outputs.has_changes == 'true'
    runs-on: ubuntu-latest
    environment: review
    strategy:
      matrix: ${{ fromJson(needs.detect-changes.outputs.matrix) }}
      fail-fast: false
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Run Cursor Code Review
        uses: Aura-app/github-actions@v1
        with:
          scope_path: ${{ matrix.path }}
          folder_guidelines_path: ${{ matrix.guidelines }}
          folder_label: ${{ matrix.label }}
          model: ${{ vars.MODEL }}
          cursor_api_key: ${{ secrets.CURSOR_API_KEY }}
```

### 4. Add your Cursor API key

Add `CURSOR_API_KEY` as a secret in your repo's `review` environment:

**Settings > Environments > review > Add secret > `CURSOR_API_KEY`**

Optionally set a `MODEL` variable in **Settings > Variables** to override the default model.

## How It Works

1. A PR is opened or updated
2. The caller workflow detects which configured scopes have changed files
3. For each changed scope, a parallel job runs the Cursor Code Review action
4. The action reads both general and folder-specific review guidelines
5. Cursor reviews the diff and posts inline comments on the PR

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `scope_path` | Yes | | Folder prefix to review (e.g. `apis/`) |
| `folder_guidelines_path` | Yes | | Path to folder-specific review guidelines |
| `folder_label` | Yes | | Human-readable label for PR comment prefixes |
| `model` | No | `gpt-5.2` | Cursor model override |
| `cursor_api_key` | Yes | | Cursor API key |
| `general_guidelines_path` | No | `REVIEW_GUIDELINES.md` | Path to repo-root general guidelines |

## License

MIT
