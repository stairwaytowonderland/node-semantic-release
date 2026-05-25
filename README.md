# node-semantic-release

A composite GitHub Action that installs dependencies, builds a Node.js project, runs [semantic-release](https://semantic-release.gitbook.io/semantic-release/),
and base64-encodes the release notes for safe downstream transport.

## Usage

```yaml
# Use PAT instead of GITHUB_TOKEN to cause semantic-release to trigger tag pipeline when tag is created.
# 'Fine-grained token' repo permissions: Contents
- uses: stairwaytowonderland/node-semantic-release@v1
  with:
    github-token: <PAT>
```

## Inputs

| Name           | Description                                                                                                                     | Required | Default |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------- | -------- | ------- |
| `github-token` | GitHub token passed to semantic-release. Requires `contents: write` and `issues: write` permissions (or a PAT with repo scope). | No¹      | `''`    |
| `node-version` | Node.js version to use.                                                                                                         | No       | `24`    |
| `build-only`   | Run the build step but skip the release step. Useful for CI checks.                                                             | No       | `false` |
| `typecheck`    | Run `npm run typecheck` and skip the release step. Useful for CI checks.                                                        | No       | `false` |

> [!NOTE]
> ¹ `github-token` is technically optional in the schema but is **required** whenever `build-only` is not `true`.
> The action will fail with a descriptive error if a token is not provided at release time.

## Outputs

| Name                       | Description                                                                                                                                                           |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `new-release-published`    | `'true'` if a new release was published; `'false'` otherwise.                                                                                                         |
| `new-release-version`      | Version number of the new release (e.g. `1.2.3`). Empty if no release was published.                                                                                  |
| `new-release-git-tag`      | Git tag of the new release (e.g. `v1.2.3`). Empty if no release was published.                                                                                        |
| `new-release-notes`        | Release notes in Markdown. Empty if no release was published.                                                                                                         |
| `new-release-notes-base64` | Base64-encoded release notes. Useful for safely passing release notes through `workflow_dispatch` inputs or environment variables. Empty if no release was published. |

## Prerequisites

Your repository must have the [`semantic-release-export-data`](https://github.com/felipecrs/semantic-release-export-data)
plugin installed and configured at the begging of the releaserc plugins list. The action checks for it at runtime and
fails with a descriptive error if it is missing.

**`package.json`**

```json
{
  "devDependencies": {
    "semantic-release": "...",
    "semantic-release-export-data": "...",
    "conventional-changelog-conventionalcommits": "...",
    "@semantic-release/changelog": "...",
    "@semantic-release/commit-analyzer": "...",
    "@semantic-release/git": "...",
    "@semantic-release/npm": "...",
    "@semantic-release/release-notes-generator": "...",
  }
}
```

**`.releaserc` (or equivalent)**

```json
{
  "plugins": [
    "semantic-release-export-data",
    "@semantic-release/changelog",
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/npm",
    "@semantic-release/git"
  ]
}
```

## Examples

### Basic release workflow

```yaml
name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - name: Run semantic release
        id: release
        uses: stairwaytowonderland/node-semantic-release@main
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Publish release
        run: gh release create "${{ steps.release.outputs.new-release-git-tag }}" \
              --repo "${{ github.repository }}" \
              --title "${{ steps.release.outputs.new-release-git-tag }}" \
              --notes-file "${{ steps.release.outputs.new-release-notes }}$"

      - if: steps.release.outputs.new-release-published == 'true'
        run: |
          echo "Created release ${{ steps.release.outputs.new-release-version }}"
```

### Build only (no release) and typecheck

> [!TIP]
> Useful for tests.

```yaml
- uses: stairwaytowonderland/node-semantic-release@main
  with:
    build-only: true
    typecheck: true
```

### Passing release notes downstream via `workflow_dispatch`

The `new-release-notes-base64` output lets you safely forward Markdown release notes — which may contain newlines, quotes,
and special characters — through `workflow_dispatch` inputs or other steps without quoting issues.

```yaml
- name: Trigger downstream workflow
  if: steps.release.outputs.new-release-published == 'true'
  uses: actions/github-script@v7
  with:
    script: |
      await github.rest.actions.createWorkflowDispatch({
        owner: context.repo.owner,
        repo: context.repo.repo,
        workflow_id: 'publish.yaml',
        ref: context.ref,
        inputs: {
          tag: '${{ steps.release.outputs.new-release-git-tag }}',
          notes-b64: '${{ steps.release.outputs.new-release-notes-base64 }}'
        }
      })
```
