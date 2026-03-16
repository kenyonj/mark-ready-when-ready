# Mark Ready When Ready

A GitHub Action that automatically marks a draft pull request as ready for
review once all required checks pass.

## How it works

1. You open a draft PR and add a trigger label (default: `Mark Ready When Ready`)
2. The action checks preconditions — if the PR isn't a draft or doesn't have the
   label, it exits immediately with `result=skipped`
3. If the repo has no required checks configured, the action skips gracefully
4. Otherwise, it watches for all required checks to complete successfully
5. It pauses briefly, then watches again to catch any late-arriving checks
6. It verifies results via the GitHub GraphQL API (paginated, handles large check suites)
7. It confirms the PR has no merge conflicts
8. If everything looks good, it marks the PR as ready for review and removes the label

This is especially useful for repos with long CI suites — open your PR as a
draft, slap on the label, and walk away. The PR will be marked ready for review
only when CI is fully green.

## Usage

```yaml
name: Mark PR Ready When Ready

on:
  pull_request:
    types: [opened, edited, labeled, unlabeled, synchronize]

permissions:
  checks: read
  contents: write
  pull-requests: write
  statuses: read

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  mark-ready:
    runs-on: ubuntu-latest
    steps:
      - uses: kenyonj/mark-ready-when-ready@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

That's it. The action checks for the trigger label and draft status
internally — if the PR isn't a draft or doesn't have the label, the action
exits immediately with `result=skipped`.

> **Important:** The workflow must include `contents: write` — without it,
> `GITHUB_TOKEN` cannot call the `markPullRequestReadyForReview` GraphQL
> mutation and will fail with `Resource not accessible by integration`.

> **Tip:** If you want to avoid allocating a runner when the preconditions
> aren't met, you can optionally add an `if:` to the job — but it's not
> required.

### Using a GitHub App token

For organizations or repos where `GITHUB_TOKEN` permissions are restricted by
policy, you can use a GitHub App token instead:

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
| `result`         | `ready`, `failing-checks`, `conflicting`, or `skipped`                                                               |
| `failing-checks` | Names of required checks that failed (empty string if all passed)                                                     |

The `result` output is `skipped` when:
- The PR is not a draft
- The trigger label is not present
- The repo has no required checks configured

## Required permissions

The workflow calling this action needs these permissions for `GITHUB_TOKEN`:

```yaml
permissions:
  checks: read
  contents: write       # required for markPullRequestReadyForReview
  pull-requests: write
  statuses: read
```

> **Note:** `contents: write` is required even though the action doesn't modify
> repository contents. Without it, `GITHUB_TOKEN` cannot call the
> `markPullRequestReadyForReview` GraphQL mutation.

## Verification strategy

The action uses a "trust but verify" approach:

1. **Precondition check** — exits early if the PR isn't a draft or the trigger
   label isn't present
2. **`gh pr checks --watch`** watches for required checks to complete (skips
   gracefully if no required checks are configured)
3. A configurable **pause** catches checks that are re-triggered or start late
4. **`gh pr checks --watch`** runs again to confirm everything is still green
5. A **GraphQL query** independently verifies that no required check suites have
   failing conclusions (`ACTION_REQUIRED`, `TIMED_OUT`, `CANCELLED`, `FAILURE`,
   `STARTUP_FAILURE`) and no required commit statuses are in a `FAILURE` state
6. The **mergeable state** is checked to ensure the PR doesn't have conflicts

This multi-layered approach prevents marking a PR as ready when there are
transient or late-arriving failures.

## License

[MIT](LICENSE)
