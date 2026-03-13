# Mark Ready When Ready

A GitHub Action that automatically marks a draft pull request as ready for
review once all required checks pass.

## How it works

1. You open a draft PR and add a trigger label (default: `Mark Ready When Ready`)
2. This action watches for all required checks to complete successfully
3. It pauses briefly, then watches again to catch any late-arriving checks
4. It verifies results via the GitHub GraphQL API (paginated, handles large check suites)
5. It confirms the PR has no merge conflicts
6. If everything looks good, it marks the PR as ready for review and removes the label

This is especially useful for repos with long CI suites — open your PR as a
draft, slap on the label, and walk away. The PR will be marked ready for review
only when CI is fully green.

## Important: token requirements

The default `GITHUB_TOKEN` **cannot** call the `markPullRequestReadyForReview`
GraphQL mutation — this is a [known GitHub platform
limitation](https://github.com/cli/cli/issues/7213). You must use a **Personal
Access Token (PAT)** or a **GitHub App token** instead.

## Usage

### With a Personal Access Token (simplest)

1. [Create a PAT](https://github.com/settings/tokens) with `repo` scope
   (classic) or `pull-requests: write` + `checks: read` + `statuses: read`
   (fine-grained)
2. Add it as a repository secret (e.g., `PAT_TOKEN`)
3. Reference it in your workflow:

```yaml
name: Mark PR Ready When Ready

on:
  pull_request:
    types: [opened, edited, labeled, unlabeled, synchronize]

permissions: {}

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  mark-ready:
    name: Mark as ready after successful checks
    runs-on: ubuntu-latest
    if: |
      contains(github.event.pull_request.labels.*.name, 'Mark Ready When Ready') &&
      github.event.pull_request.draft == true
    steps:
      - name: Mark ready when ready
        uses: kenyonj/mark-ready-when-ready@v1
        with:
          github-token: ${{ secrets.PAT_TOKEN }}
```

### With a GitHub App token (recommended for organizations)

GitHub App tokens are scoped, rotated automatically, and don't tie to a
personal account — ideal for shared repos and orgs.

```yaml
    steps:
      - name: Generate token
        id: generate-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Mark ready when ready
        uses: kenyonj/mark-ready-when-ready@v1
        with:
          github-token: ${{ steps.generate-token.outputs.token }}
```

## Inputs

| Input            | Required | Default                  | Description                                                                |
| ---------------- | -------- | ------------------------ | -------------------------------------------------------------------------- |
| `github-token`   | Yes      | —                        | Token with permission to read checks and write to pull requests            |
| `label`          | No       | `Mark Ready When Ready`  | The label that triggers the action                                         |
| `pause-seconds`  | No       | `20`                     | Seconds to pause between verification rounds (catches late-arriving checks) |
| `remove-label`   | No       | `true`                   | Whether to remove the trigger label after marking the PR ready             |

## Outputs

| Output           | Description                                                           |
| ---------------- | --------------------------------------------------------------------- |
| `result`         | `ready`, `failing-checks`, `conflicting`, or `skipped`                |
| `failing-checks` | Names of required checks that failed (empty string if all passed)     |

## Required token permissions

The token provided to `github-token` needs these scopes / permissions:

| Scope                  | Why                                              |
| ---------------------- | ------------------------------------------------ |
| `checks: read`         | Watch check run status                           |
| `statuses: read`       | Verify commit status contexts                    |
| `pull-requests: write` | Mark the PR ready and manage labels              |
| `contents: read`       | Access repository data via the GraphQL API       |

> **Note:** `GITHUB_TOKEN` cannot be used — see
> [token requirements](#important-token-requirements) above.

## Verification strategy

The action uses a "trust but verify" approach:

1. **`gh pr checks --watch`** watches for required checks to complete
2. A configurable **pause** catches checks that are re-triggered or start late
3. **`gh pr checks --watch`** runs again to confirm everything is still green
4. A **GraphQL query** independently verifies that no required check suites have
   failing conclusions (`ACTION_REQUIRED`, `TIMED_OUT`, `CANCELLED`, `FAILURE`,
   `STARTUP_FAILURE`) and no required commit statuses are in a `FAILURE` state
5. The **mergeable state** is checked to ensure the PR doesn't have conflicts

This multi-layered approach prevents marking a PR as ready when there are
transient or late-arriving failures.

## License

[MIT](LICENSE)
