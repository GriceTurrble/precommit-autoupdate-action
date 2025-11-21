# precommit-autoupdate-action

Runs [pre-commit](https://pre-commit.com) autoupdate within GitHub Actions,
creating a Pull Request automatically with any updates detected.
Use this in a scheduled job to keep your hooks up-to-date with latest changes.

## Usage

Include `precommit-autoupdate-action` in a GitHub Actions workflow.
For example:

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
      - uses: actions/checkout@v6
      - uses: griceturrble/precommit-autoupdate-action@v3
```

### Action inputs

All inputs are optional.
Many of these inputs are passed unchanged to [peter-evans/create-pull-request](https://github.com/peter-evans/create-pull-request/);
please refer there for more detailed instructions:

| Name                 | Description                                                                                                                          | Default                       |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------- |
| `token`              | Token to use for authenticating to GitHub.                                                                                           | `GITHUB_TOKEN`                |
| `python_version`     | Version of Python to use to install pre-commit.                                                                                      | `"3.14"`                      |
| `pre_commit_version` | Version of Pre-commit to use.                                                                                                        | `"4.4.0"`                     |
| `path_to_config`     | Path to the `.pre-commit-config.yaml` file.                                                                                          | `".pre-commit-config.yaml"`   |
| `commit_message`     | Message to use for the generated commit.                                                                                             | `"ci: pre-commit autoupdate"` |
| `pr_title`           | Title of the PR created by this action (passed as `title` to `create-pull-request`).                                                 | `"pre-commit autoupdate"`     |
| `pr_branch_name`     | git branch that is pushed with these changes (passed as `branch` to `create-pull-request`).                                          | `"ci/pre-commit-autoupdate"`  |
| `create_as_draft`    | Whether to create the PR in draft mode (passed as `draft` to `create-pull-request`).                                                 | `"false"`                     |
| `pr_labels`          | Comma- or newline-separated list of labels to add to the pull request (passed as `labels to `create-pull-request`).                  |                               |
| `pr_assignees`       | Comma- or newline-separated list of GitHub usernames to assign to the pull request (passed as `assignees` to `create-pull-request`). |                               |
| `pr_reviewers`       | Comma- or newline-separated list of GitHub usernames to request reviews from (passed as `reviewers` to `create-pull-request`).       |                               |
| `pr_team_reviewers`  | Comma- or newline-separated list of GitHub team slugs to request reviews from (passed as `team-reviewers` to `create-pull-request`). |                               |

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

### Workaround: create PRs as draft

> [!note]
> Draft PRs may not be available to you.
> See [GitHub Docs](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/changing-the-stage-of-a-pull-request)
> for more details.

One manual option for working around the GH Actions limitations
is to use the `create_as_draft` arg for this action
(which is passed to the `draft` arg in the create-pull-request action):

```yaml
# precommit-autoupdate.yaml
- uses: griceturrble/precommit-autoupdate-action@v3
  with:
    create_as_draft: true
```

In your CI workflow,
include the `ready_for_review` type as a trigger:

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

### Workaround: edit PR titles

Another manual workaround option is to create PRs with a "draft" title,
then editing the title of that PR.
This is handy when working in a repo that does not permit draft PRs.

Simply change the `pr_title` arg for this action to include
some prefix, such as `[Draft]`:

```yaml
# precommit-autoupdate.yaml
- uses: griceturrble/precommit-autoupdate-action@v3
  with:
    pr_title: "[Draft] pre-commit autoupdate"
```

In your CI workflow,
include the `edited` type as a trigger:

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
      - edited
```

With these changes, new PRs will be opened with the title `[Draft] pre-commit autoupdate`.
You may then open the new PR and manually change its title,
such as removing the `[Draft]` prefix.
This triggers the `edited` event for the PR,
which in turn will trigger your CI workflow.

Note that the `edited` type is also triggered
whenever a PR description is updated, as well.
This may impact the workflow of your other PRs,
where any change (including clicking task checkboxes) counts as an "edit"
that will trigger a new CI run.

If you don't already,
consider setting the following options in your workflow to prevent extra GitHub Actions usage:

- Use [workflow concurrency](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs) to cancel in-flight jobs when a new one begins,
  such as:

  ```yaml
  concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: true
  ```

- Use the same `[Draft]` title prefix as part of your other CI workflows,
  skipping entire jobs if this text is present in that title:

  ```yaml
  jobs:
    ci:
      runs-on: ubuntu-latest
      if: ${{ !github.event.pull_request || !contains(github.event.pull_request.title, '[draft]') }}
  ```

  See [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#jobsjob_idif)
  for more details.
