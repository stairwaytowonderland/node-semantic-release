# node-semantic-release

A composite GitHub Action that installs dependencies, builds a Node.js project, runs [semantic-release](https://semantic-release.gitbook.io/semantic-release/),
and base64-encodes the release notes for safe downstream transport.

The action operates in two distinct modes controlled by the `publish` input:

- **Release mode** (default): Sets up Node.js, installs dependencies, builds the project, runs semantic-release, and
optionally commits release assets and creates a git tag.
- **Publish mode** (`publish: true`): Skips all build and tag creation steps, and creates a GitHub Release from an existing
tag, using either base64-encoded release notes or release notes determine from the tag.

## Usage

```yaml
# Recommended: use a PAT so that the created tag triggers downstream tag-push workflows.
# Fine-grained token permissions required: Contents (read/write)
- uses: stairwaytowonderland/node-semantic-release@v1
  with:
    github-token: ${{ secrets.GH_PAT }}
```

## Token Behavior

> [!IMPORTANT]
> The choice of token has a significant effect on whether the GitHub release is created automatically.

When a git tag is created by a workflow using **`secrets.GITHUB_TOKEN`**, GitHub deliberately does not fire
additional workflow runs for that tag push (to prevent infinite loops). This means a separate `push: tags`
publish workflow will **not** be triggered.

To work around this, choose one of the following approaches:

1. **Use a Personal Access Token (PAT) — recommended.** Tags pushed with a PAT do trigger downstream
   workflows normally. Grant the fine-grained token **Contents** read/write permission.

2. **Use `secrets.GITHUB_TOKEN` + `workflow_dispatch` for publish.** Run semantic-release on push to
   `main`, then dispatch a separate publish workflow from the same job using the
   `new-release-notes-base64` output. See the [workflow_dispatch example](#using-secretsgithub_token-with-workflow_dispatch-publish)
   below.

3. **Use `secrets.GITHUB_TOKEN` + the `@semantic-release/github` plugin.** Configure
   `@semantic-release/github` in your `.releaserc` and semantic-release will create the GitHub Release
   itself during the release run — no separate publish step required.

## Permissions

When using `secrets.GITHUB_TOKEN` the calling job must declare the following permissions:

```yaml
permissions:
  contents: write
  issues: write
  pull-requests: write
```

When using a PAT these job-level permissions are not required, but the token itself must have
**Contents** read/write scope (fine-grained) or the `repo` scope (classic).

The publish job only needs `contents: write`.

## Inputs

### Release mode inputs

> [!NOTE]
> **`manual-tag` mode** runs semantic-release in dry-run to determine the next version and generate release
> notes, then manually commits the specified assets (`changelog-file`, `additional-assets-json`) and pushes
> an annotated git tag. This is useful for debugging.

| Name                     | Description                                                                                                                     | Required | Default                              |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------------------ |
| `node-version`           | Node.js version to use.                                                                                                         | No       | `24`                                 |
| `github-token`           | GitHub token passed to semantic-release. Requires `contents: write` and `issues: write` permissions (or a PAT with repo scope). | No¹      | `${{ github.token }}`                |
| `build-only`             | Run the build step but skip the release step. Useful for CI checks.                                                             | No       | `''`                                 |
| `typecheck`              | Run `npm run typecheck` after the build and skip the release step. Useful for CI checks.                                        | No       | `''`                                 |
| `dry-run`                | Run semantic-release in `--dry-run` mode. No tag or release is created.                                                         | No       | `''`                                 |
| `manual-tag`             | Compute the next version with semantic-release (dry-run), then commit release assets and push the tag manually. See note below. | No       | `''`                                 |
| `changelog-file`         | Path to the changelog file to commit when using `manual-tag` mode (e.g. `CHANGELOG.md`).                                        | No²      | `CHANGELOG.md`                       |
| `additional-assets-json` | JSON array of additional files to commit when using `manual-tag` mode (e.g. `["package.json","package-lock.json"]`).            | No²      | ["package.json","package-lock.json"] |
| `commit-author`          | Git author name for commits made by the action when using `manual-tag` mode.                                                    | No²      | `semantic-release-bot`               |
| `commit-email`           | Git author email for commits made by the action when using `manual-tag` mode.                                                   | No²      | `semantic-release-bot@martynus.net`  |
| `commit-prefix`          | Prefix for the release commit message when using `manual-tag` mode.                                                             | No²      | `chore(release):`                    |

### Publish mode inputs

Set `publish: true` to activate publish mode. All release-mode build/release steps are skipped.

| Name           | Description                                                                              | Required | Default                  |
| -------------- | ---------------------------------------------------------------------------------------- | -------- | ------------------------ |
| `publish`      | Set to `true` to create a GitHub Release from an existing tag.                           | No³      | `''`                     |
| `tag`          | The git tag to publish (e.g. `v1.2.3`).                                                  | No³      | `${{ github.ref_name }}` |
| `notes-b64`    | Base64-encoded release notes. If omitted, notes are read from the annotated tag message. | No³      | `''`                     |
| `github-token` | Token used by `gh release create`. Requires `contents: write`.                           | No¹      | `${{ github.token }}`    |

> [!NOTE]
> ¹ `github-token` is technically optional but is **required whenever `build-only` is *not* `true`**.
>
> ² Only relevant when `manual_tag` is `true`.
>
> ³ Only relevant when `publish` is `true`.

## Outputs

### Release mode outputs

| Name                       | Description                                                                                                                                                           |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `new-release-published`    | `true` if a new release was published; `false` otherwise.                                                                                                             |
| `new-release-version`      | Version number of the new release (e.g. `1.2.3`). Empty if no release was published.                                                                                  |
| `new-release-git-tag`      | Git tag of the new release (e.g. `v1.2.3`). Empty if no release was published.                                                                                        |
| `new-release-notes`        | Release notes in Markdown. Empty if no release was published.                                                                                                         |
| `new-release-notes-base64` | Base64-encoded release notes. Useful for safely passing release notes through `workflow_dispatch` inputs or environment variables. Empty if no release was published. |

### Publish mode outputs

| Name                | Description                                            |
| ------------------- | ------------------------------------------------------ |
| `release-url`       | URL of the GitHub Release that was created.            |
| `release-tag`       | Git tag of the release that was published.             |
| `release-sha`       | Git SHA of the release tag that was published.         |
| `release-notes-b64` | Base64-encoded release notes of the published release. |
| `release-published` | `true` if the release was published.                   |

## Prerequisites

The [`semantic-release-export-data`](https://github.com/felipecrs/semantic-release-export-data) plugin must
be installed and listed **first** in your `.releaserc` plugins array. The action verifies this at runtime and
fails with a descriptive error if it is missing.

If neither a `package.json` nor a `.releaserc` (or equivalent) is found in the workspace, the action copies
its own default templates from the `templates/` directory. It is recommended to provide your own.

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
    "@semantic-release/release-notes-generator": "..."
  }
}
```

**`.releaserc` (or equivalent)**

```json
{
  "branches": ["main"],
  "plugins": [
    "semantic-release-export-data",
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/npm", { "npmPublish": false }],
    [
      "@semantic-release/git",
        {
          "assets": ["CHANGELOG.md", "package.json", "package-lock.json"],
          "message": "chore(release): ${nextRelease.version}\n\n${nextRelease.notes}"
        }
    ]
  ]
}
```

## Examples

### Recommended: PAT with a separate publish workflow

A PAT ensures the pushed tag triggers a downstream `push: tags` workflow automatically.

```yaml
# .github/workflows/release.yaml
name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GH_PAT }}

      - name: Run semantic release
        id: release
        uses: stairwaytowonderland/node-semantic-release@v1
        with:
          github-token: ${{ secrets.GH_PAT }}
```

```yaml
# .github/workflows/publish.yaml
name: Publish

on:
  push:
    tags: ['v*']

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.ref_name }}
          fetch-depth: 0

      - name: Create GitHub Release
        uses: stairwaytowonderland/node-semantic-release@v1
        with:
          publish: true
          github-token: ${{ secrets.GH_PAT }}
          tag: ${{ github.ref_name }}
```

### Using `secrets.GITHUB_TOKEN` with `workflow_dispatch` publish

When you cannot use a PAT, run the release and dispatch the publish step from the same job.
The `new-release-notes-base64` output safely transports Markdown release notes — which may contain
newlines, quotes, and special characters — through `workflow_dispatch` inputs.

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
      actions: write    # required to dispatch the publish workflow
      contents: write
      issues: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run semantic release
        id: release
        uses: stairwaytowonderland/node-semantic-release@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          manual-tag: true
          changelog-file: CHANGELOG.md
          additional-assets-json: '["package.json","package-lock.json"]'

      - name: Dispatch publish workflow
        if: steps.release.outputs.new-release-published == 'true'
        uses: actions/github-script@v7
        env:
          TAG: ${{ steps.release.outputs.new-release-git-tag }}
          NOTES_B64: ${{ steps.release.outputs.new-release-notes-base64 }}
        with:
          script: |
            await github.rest.actions.createWorkflowDispatch({
              owner: context.repo.owner,
              repo: context.repo.repo,
              workflow_id: 'publish.yaml',
              ref: process.env.TAG,
              inputs: {
                tag: process.env.TAG,
                'notes-b64': process.env.NOTES_B64
              }
            })
```

```yaml
# .github/workflows/publish.yaml — triggered by workflow_dispatch from the release job above
name: Publish

on:
  workflow_dispatch:
    inputs:
      tag:
        description: 'Tag to release (e.g. v1.2.3)'
        type: string
        required: true
      notes-b64:
        description: 'Base64-encoded release notes'
        type: string
        required: false

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.tag }}
          fetch-depth: 0

      - name: Create GitHub Release
        uses: stairwaytowonderland/node-semantic-release@v1
        with:
          publish: true
          github-token: ${{ secrets.GITHUB_TOKEN }}
          tag: ${{ inputs.tag }}
          notes-b64: ${{ inputs.notes-b64 }}
```

### Build only (no release) and typecheck

> [!TIP]
> Useful for pull request CI checks.

```yaml
- uses: stairwaytowonderland/node-semantic-release@v1
  with:
    build-only: true
    typecheck: true
```

### Dry run

```yaml
- uses: stairwaytowonderland/node-semantic-release@v1
  with:
    github-token: ${{ secrets.GH_PAT }}
    dry-run: true
```
