# Mark Ready When Ready

[![GitHub release](https://img.shields.io/github/v/release/kenyonj/mark-ready-when-ready)](https://github.com/kenyonj/mark-ready-when-ready/releases/latest)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![GitHub Marketplace](https://img.shields.io/badge/marketplace-mark--ready--when--ready-blue?logo=github)](https://github.com/marketplace/actions/mark-ready-when-ready)
[![Sponsor](https://img.shields.io/badge/sponsor-♥-pink?logo=github)](https://github.com/sponsors/kenyonj)

A GitHub Action that automatically marks a draft pull request as ready for
review once all required checks pass.

## How it works

1. You open a draft PR and add a trigger label (default: `Mark Ready When Ready`)
2. The action validates that the token has all required permissions upfront — if
   anything is missing, it fails immediately with a clear error showing the exact
   `permissions:` block to add
3. It checks preconditions — if the PR isn't a draft or doesn't have the
   label, it exits immediately with `result=skipped`
4. If the repo has no required checks configured, the action skips gracefully
5. Otherwise, it watches for all required checks to complete successfully
6. It pauses briefly, then watches again to catch any late-arriving checks
7. It verifies results via the GitHub GraphQL API (paginated, handles large check suites)
8. It confirms the PR has no merge conflicts
9. If everything looks good, it marks the PR as ready for review and removes the label

This is especially useful for repos with long CI suites — open your PR as a
draft, slap on the label, and walk away. The PR will be marked ready for review
only when CI is fully green.

## Usage

```yaml
name: Mark PR Ready When Ready

on:
  pull_request:
    types: [opened, edited, labeled, unlabeled, synchronize]

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  mark-ready:
    runs-on: ubuntu-latest
    permissions:
      checks: read
      contents: write
      pull-requests: write
      statuses: read
    steps:
      - uses: kenyonj/mark-ready-when-ready@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

That's it. The action validates permissions upfront, checks for the trigger
label and draft status internally — if anything isn't right, you'll get a
clear error message or the action exits with `result=skipped`.

> **Important:** The workflow must include `contents: write` at the **job
> level** — without it, `GITHUB_TOKEN` cannot call the
> `markPullRequestReadyForReview` GraphQL mutation and will fail with
> `Resource not accessible by integration`. As of v1.1.2, the action validates
> permissions upfront and will tell you exactly what's missing.

### Saving runner costs with a job-level guard

The action checks for the trigger label and draft status internally, so a
job-level `if:` is not strictly required. However, without it GitHub still
allocates a runner, boots it, and downloads the action before the
precondition check can exit — that startup cost adds up on busy repos.

Adding an `if:` to the job lets GitHub evaluate the condition from the event
payload **before** allocating a runner, skipping the job entirely at zero
cost:

```yaml
jobs:
  mark-ready:
    runs-on: ubuntu-latest
    permissions:
      checks: read
      contents: write
      pull-requests: write
      statuses: read
    if: |
      contains(github.event.pull_request.labels.*.name, 'Mark Ready When Ready') &&
      github.event.pull_request.draft == true
    steps:
      - uses: kenyonj/mark-ready-when-ready@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

This is recommended for repos with frequent PR activity. For smaller repos
the difference is negligible and you can omit the `if:` for simplicity.

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
>
> As of v1.1.2, the action validates these permissions at the start of every run.
> If any are missing, it fails immediately with a clear error message showing the
> exact `permissions:` block to add to your workflow. Permissions should be set at
> the **job level**, not the top-level workflow `permissions:` key.

## Verification strategy

The action uses a "trust but verify" approach:

1. **Permission validation** — verifies the token has all required permissions
   (`contents:write`, `pull-requests:write`, `checks:read`, `statuses:read`)
   and fails fast with an actionable error message if any are missing
2. **Precondition check** — exits early if the PR isn't a draft or the trigger
   label isn't present
3. **`gh pr checks --watch`** watches for required checks to complete (skips
   gracefully if no required checks are configured)
4. A configurable **pause** catches checks that are re-triggered or start late
5. **`gh pr checks --watch`** runs again to confirm everything is still green
6. A **GraphQL query** independently verifies that no required check suites have
   failing conclusions (`ACTION_REQUIRED`, `TIMED_OUT`, `CANCELLED`, `FAILURE`,
   `STARTUP_FAILURE`) and no required commit statuses are in a `FAILURE` state
7. The **mergeable state** is checked to ensure the PR doesn't have conflicts

This multi-layered approach prevents marking a PR as ready when there are
transient or late-arriving failures.

## License

[MIT](LICENSE)
