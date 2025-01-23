# precommit-autoupdate-action

Runs [pre-commit](https://pre-commit.com) autoupdate within GitHub Actions,
creating a Pull Request automatically with any updates detected.
Use this in a scheduled job to keep your hooks up-to-date with latest changes.

## Usage

Include `precommit-autoupdate-action` in a GitHub Actions workflow:

```yaml
# .github/workflows/precommit-autoupdate.yaml
name: pre-commit autoupdate

on:
  # To call the workflow on-demand
  workflow_dispatch:
  # To have it run automatically on a schedule:
  schedule:
    # Example: every Monday at 11AM UTC (~6AM US/Eastern)
    - cron: "0 11 * * 1"

jobs:
  pre-commit-autoupdate:
    permissions:
      # The workflow must be able to write changes
      # to `.pre-commit-config.yaml`:
      contents: write
      # It must also have permission to create PRs:
      pull-requests: write
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: griceturrble/precommit-autoupdate-action@v1
        with:
          # All arguments below are optional
          # and are shown with their default values:

          # token to use for authenticating to GitHub.
          token: ${{ github.token }}
          # Version of Python to use to install pre-commit.
          python_version: "3.12"
          # Version of Pre-commit to install.
          pre_commit_version: "4.1.0"
          # Path to the .pre-commit-config.yaml file.
          path_to_config: ".pre-commit-config.yaml"
          # Title of the PR created:
          pr_title: "pre-commit autoupdate"
          # git branch that is pushed with these changes.
          pr_branch_name: "ci/pre-commit-autoupdate"
          # Whether to create the PR in draft mode.
          create_as_draft: false
```

## Permissions

> [!note]
> This action wraps
> [peter-evans/create-pull-request](https://github.com/peter-evans/create-pull-request/)
> to create a pull request.
> The documentation of that action has extensive details
> about setting up the correct workflow permissions.
> Below is only a short summary to get you started.

This workflow requires additional permissions
and has a key limitation that you will need to work around
if you want the automatic PRs to trigger additional CI workflows.

1. As shown in the example above,
   your workflow must, at minimum,
   grant `contents: write` and `pull-requests: write` permissions,
   either at the workflow or job levels.

2. You _must_ explicitly allow GitHub Actions to create pull requests
   for this action to work.
   In your own repo,
   under **Settings > Actions > General**,
   scroll to the bottom of the page and turn on the
   **"Allow GitHub Actions to create and approve pull requests"** checkbox.

3. When using the default `token` argument (your `GITHUB_TOKEN`),
   the PR that is generated _cannot_ trigger additional GitHub Actions workflows.
   This is a limitation set up by GitHub themselves.

To work around the `token` limitation and have the new PR trigger workflows,
see one of these available options in the
[peter-evans/create-pull-request docs](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#triggering-further-workflow-runs).
These may include creating a repo-scoped PAT,
creating SSH Deploy keys to push the branch,
or creating a GitHub App.

Another, more manual option,
is to use the `create_as_draft` arg for this action
(which is passed to the `draft` arg in the create-pull-request action):

```yaml
# precommit-autoupdate.yaml
- uses: griceturrble/precommit-autoupdate-action@v1
  with:
    create_as_draft: true
```

You can then update your CI workflow for standard pull requests
to trigger on the `ready_for_review` type:

```yaml
# ci.yaml
on:
  pull_request:
    types:
      # These types are the defaults for a `pull_request` trigger:
      - opened
      - synchronize
      - reopened
      # Include all the above AND this one:
      - ready_for_review
```

With this in place, `precommit-autoupdate-action` will generate
draft PRs only.
You may then open the new PR and manually mark it "Ready for review".
This manual action will then trigger your CI workflow.
