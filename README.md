# Signal-58 organisation defaults

Default community health files and shared workflows for repositories in the Signal-58 GitHub organisation. GitHub uses a file from here for any repository that doesn't have its own.

- `pull_request_template.md`: the default pull request description. Its `OP#` line links the pull request to an OpenProject work package.
- `.github/workflows/promote.yml`: the shared Promote workflow that releases `staging` to production. See below.

This repository has to be public for GitHub to apply the defaults. Keep it to templates and shared workflows: nothing private belongs here. Workflows take secrets from the calling repository; never put a secret, token or private URL in this repository.

## Promote: releasing staging to production

Repositories that deploy `staging` to a staging site and `main` to production release by fast-forwarding `main` to a tested `staging` commit. Nobody merges into `main`, so it never gains a commit that `staging` lacks, and release pull requests and `main` → `staging` sync pull requests aren't needed.

### What the workflow checks

Promote refuses to release unless all of these hold, in this order:

1. The person who started it is an admin of the repository.
2. The commit is on `staging` (the latest `staging` commit when no commit is given).
3. Every check that `main`'s rulesets require has passed on that exact commit (`success`, `neutral` or `skipped`).
4. `main` can fast-forward to it. An older commit is refused, so a release is undone by reverting on `staging` and promoting again.

Then it moves `main`, which starts the production deploy, and creates a GitHub Release named `release-YYYY.MM.DD-N` that lists the pull requests released. A dry run does every check and lists the pull requests without moving `main`.

### What a repository needs

- **CI on pushes to `staging`**, not only on pull requests. Promote checks the commit that is actually on `staging`. A merge creates a new commit, and a pull-request run tests a different one.
- **The Signal58 Promote GitHub App** installed on the repository, with the organisation variable `PROMOTE_APP_CLIENT_ID` and secret `PROMOTE_APP_PRIVATE_KEY` available to it.
- **Rulesets:**
  - `staging`: require a pull request and the CI checks; block force pushes and deletion.
  - `main`: require the CI checks; block force pushes and deletion. Nobody bypasses.
  - `main`: restrict updates, with only the Signal58 Promote app allowed to bypass.
- **A caller workflow** at `.github/workflows/promote.yml` on the default branch:

```yaml
name: Promote

on:
  workflow_dispatch:
    inputs:
      sha:
        description: Staging commit to release (empty releases the latest)
        required: false
        default: ''
      dry_run:
        description: Check only, don't release
        type: boolean
        default: false

permissions:
  contents: read

jobs:
  promote:
    uses: Signal-58/.github/.github/workflows/promote.yml@v1
    with:
      app_client_id: ${{ vars.PROMOTE_APP_CLIENT_ID }}
      sha: ${{ inputs.sha }}
      dry_run: ${{ inputs.dry_run }}
    secrets:
      PROMOTE_APP_PRIVATE_KEY: ${{ secrets.PROMOTE_APP_PRIVATE_KEY }}
```

To release: Actions → Promote → Run workflow.

### Versions

Callers use the `v1` tag. Fixes that keep the inputs and behaviour move `v1`; a change that callers must adapt to gets `v2`.
