# Reusable workflows

Two reusable workflows live in `.github/workflows/` here. Every setting is in the reusable file; a
repository adopts one by adding the small caller below. Auto-merge defaults to the isolated
self-hosted `ci-control-merge` pool; stale-PR maintenance keeps the `general` pool. GitHub-hosted
runners are not used by private repositories. A caller can choose another pool with the optional
`runner-labels` input, a JSON label array. There is no fallback if the selected pool is offline.
Auto-merge requires `gh`, `jq`, Python 3 and PyYAML on its control workers. A **public** caller must
pass `runner-labels: '["ubuntu-latest"]'` explicitly: public-repository code must never run on our
own machines, and public repositories do not consume hosted minutes.

For either reusable workflow, add alongside `uses`:

```yaml
with:
  runner-labels: '["self-hosted", "Linux", "X64", "db-capable"]'
```


## Stale pull requests

`stale-prs.yml` marks a pull request `stale` after 14 days without activity and closes it 16 days
later, about 30 days in all. Any new activity removes the label. The `keep-open` label exempts a
pull request, as do drafts and Dependabot pull requests (label `dependencies`). Branches are never deleted, so a closed pull request can be reopened. Issues are not
touched.

`.github/workflows/stale.yml` in the adopting repository:

```yaml
name: Stale pull requests

on:
  schedule:
    - cron: "17 6 * * 1"
  workflow_dispatch:

jobs:
  stale:
    uses: Flowtly-PSA/.github/.github/workflows/stale-prs.yml@main
    permissions:
      actions: write
      issues: write
      pull-requests: write
```

## Auto-merge

`dependabot-auto-merge.yml` (the name predates the second path) enables auto-merge (squash) on two
kinds of pull request:

- a Dependabot pull request whose highest update is minor or patch. That covers security updates and
  grouped pull requests on the same terms; a semver-major update is never auto-merged.
- any other non-draft pull request whose author (not whoever pushed last, and not a bot) holds
  write, maintain or admin on the repository, read live from the API rather than from the event's
  `author_association`, which hides private org membership; from a branch of the same repository, except a
  `steward/*` branch, which steward merges itself, or a promotion (a `preprod` or
  `promote/*` head), which `/flowtly:release` merges as a merge commit.

Converting a pull request to draft disables its auto-merge, so draft means "do not merge". It refuses,
with a warning in the run log, when the base branch has no required status check in branch protection
or a ruleset, because `gh pr merge --auto` would then merge at once, before CI. On the trusted path
it also refuses a pull request that is already mergeable while a check on its head was skipped (CI
skipped while it was a draft); the next push arms it.

GitHub's auto-merge waits only for required checks, so a non-required check that fails after arming
would not stop the merge. The job therefore arms only once every workflow run on the head has
finished (up to about an hour) and none of its jobs failed, was cancelled or timed out, as steward's
`merge.py` refuses a red check. Each workflow is judged per event by its newest run that did not skip
outright, at its latest attempt, so a draft-era run or a superseded one is not the verdict. A head
with no workflow run is refused too. A red check refuses with a warning; a
re-run that turns it green does not re-arm (no `pull_request` event fires), so push again or merge by
hand. Only GitHub Actions jobs are seen: commit statuses and other apps' check runs are out of scope,
because callers grant no `checks: read`. A merge queue works as before: arming a green pull request
enqueues it.

Every pull request it handles holds a runner while its CI runs, and an armed one keeps
it until the merge, up to about an hour more, when a deploy workflow would run on the merge (below).
These waits use the isolated `ci-control-merge` pool so they cannot occupy the general runners
whose checks they await. Self-hosted execution adds no GitHub-hosted runner minutes.

The organization pool has two services on the existing marketing host, each with a 512 MiB
memory soft limit, 1 GiB hard limit, 128 MiB swap cap and half a CPU quota. They carry neither
`general` nor `marketing`. The **CI merge control** runner group excludes public repositories
and permits only jobs defined by this reusable workflow at `refs/heads/main`. The existing
backend repository-scoped control runner remains available to that repository. All workers
provide `gh`, `jq`, Python 3 and PyYAML; no checkout of caller code is needed.

Changing the runner pool preserves the all-checks gate, trusted-author and Dependabot rules,
head-SHA match, draft handling, and post-merge deployment dispatch. The input override still
allows a caller with different infrastructure to select its own pool.

Known exception: `flowtly-subscriptions` gates its caller behind a Dependabot-only BP-24 job, so its
other pull requests are never auto-merged. That is deliberate (fail-safe).

Auto-merge adds no privilege: no base branch requires an approving review, so anyone with write access
could already merge a green pull request by hand. It does make that silent: a push by another writer to
an armed pull request merges on green without the author seeing it. Where that is not acceptable,
require approving reviews on the base branch; auto-merge then waits for them.

The repository must allow auto-merge (Settings, General, "Allow auto-merge"), or the merge step
fails.

After enabling auto-merge the job waits (up to about an hour) for the merge, then dispatches each
deploy workflow the push would have run. A merge made with `GITHUB_TOKEN` creates no push run, so
without this step the default branch moves and production does not (flowtly-orchestration#947).
`workflow_dispatch` is exempt from that rule, so the deploy workflow must declare
`workflow_dispatch:` among its triggers. Without it, auto-merge still works and the run log carries
a warning naming what is missing. The caller must grant `actions: write`: without it the job cannot
read the head's workflow runs and arms nothing.

`.github/workflows/dependabot-auto-merge.yml` in the adopting repository:

```yaml
name: Dependabot auto-merge

on:
  pull_request:
    types: [opened, reopened, synchronize, ready_for_review, converted_to_draft]

jobs:
  auto-merge:
    uses: Flowtly-PSA/.github/.github/workflows/dependabot-auto-merge.yml@main
    permissions:
      contents: write
      pull-requests: write
      actions: write
```

## `dependabot.yml` template

`.github/dependabot.yml`; list every directory that holds a `package.json` under `directories`.

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directories:
      - "/"
    schedule:
      interval: monthly
    groups:
      security:
        applies-to: security-updates
        patterns: ["*"]
      minor-patch:
        applies-to: version-updates
        patterns: ["*"]
        update-types: [minor, patch]
      major:
        applies-to: version-updates
        patterns: ["*"]
        update-types: [major]

  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: monthly
    groups:
      github-actions:
        patterns: ["*"]
```
