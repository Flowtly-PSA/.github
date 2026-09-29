# Reusable workflows

Two reusable workflows live in `.github/workflows/` here. Every setting is in the reusable file; a
repository adopts one by adding the small caller below, unchanged. Both reusable jobs run on
`ubuntu-latest` for the reasons given in their comments (`CI_COST_CONVENTIONS.md`, rule 8).

## Stale pull requests

`stale-prs.yml` marks a pull request `stale` after 14 days without activity and closes it 16 days
later, about 30 days in all. Any new activity removes the label. The `keep-open` label exempts a
pull request. Branches are never deleted, so a closed pull request can be reopened. Issues are not
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

## Dependabot auto-merge

`dependabot-auto-merge.yml` enables auto-merge (squash) on a Dependabot pull request when its
highest update is minor or patch. That covers security updates and grouped pull requests on the
same terms; a semver-major update is never auto-merged. It also refuses, with a warning in the run
log, when the base branch has no required status check in branch protection or a ruleset, because
`gh pr merge --auto` would then merge at once, before CI.

The repository must allow auto-merge (Settings, General, "Allow auto-merge"), or the merge step
fails.

`.github/workflows/dependabot-auto-merge.yml` in the adopting repository:

```yaml
name: Dependabot auto-merge

on:
  pull_request:

jobs:
  auto-merge:
    uses: Flowtly-PSA/.github/.github/workflows/dependabot-auto-merge.yml@main
    permissions:
      contents: write
      pull-requests: write
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
