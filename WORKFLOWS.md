# Reusable workflows

Two reusable workflows live in `.github/workflows/` here. Every setting is in the reusable file; a
repository adopts one by adding the small caller below, unchanged. Both reusable jobs run on
`ubuntu-latest` for the reasons given in their comments (`CI_COST_CONVENTIONS.md`, rule 8).

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
- any other non-draft pull request whose author is OWNER, MEMBER or COLLABORATOR
  (`author_association`, not whoever pushed last), from a branch of the same repository, except a
  `steward/*` branch, which steward merges itself, or a promotion (a `preprod` or
  `promote/*` head), which `/flowtly:release` merges as a merge commit.

Converting a pull request to draft disables its auto-merge, so draft means "do not merge". It refuses,
with a warning in the run log, when the base branch has no required status check in branch protection
or a ruleset, because `gh pr merge --auto` would then merge at once, before CI. On the trusted path
it also refuses a pull request that is already mergeable while a check on its head was skipped (CI
skipped while it was a draft); the next push arms it.

GitHub's auto-merge waits only for required checks, so a non-required check that fails after arming
would not stop the merge. The job therefore arms only once every workflow run on the head has
finished (up to about an hour) and none of its jobs failed, was cancelled or timed out, judging each
job name by its newest attempt, as steward's `merge.py` does. A red check refuses with a warning; a
re-run that turns it green does not re-arm (no `pull_request` event fires), so push again or merge by
hand. Only GitHub Actions jobs are seen: commit statuses and other apps' check runs are out of scope,
because callers grant no `checks: read`. A merge queue works as before: arming a green pull request
enqueues it.

Every pull request it handles holds an `ubuntu-latest` job while its CI runs, and an armed one keeps
it until the merge, up to about an hour more, when a deploy workflow would run on the merge (below).
All of it is billed.

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
`workflow_dispatch:` among its triggers, and the caller must grant `actions: write`. Without either,
auto-merge still works and the run log carries a warning naming what is missing.

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
