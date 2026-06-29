# GitHub Actions cost conventions

**Read this before adding or editing any `.github/workflows/*.yml`.**

Every active Flowtly repo is **private**, so *every* Actions minute is billed and
rounded up to the whole minute **per job**. Parallel/matrix jobs each bill
separately, so a 3-job workflow that runs 16 min wall-clock bills ~48 min. The
cheapest run is the one that never needed to happen. Optimise for *fewer runs*
first, *faster runs* second.

## Rules for any new or edited workflow

1. **Trigger on `pull_request`, never bare `on: push`.** `on: push` fires on
   every commit to every branch (including throwaway/WIP branches) and on the
   post-merge push too — usually double the runs you want. Use:
   ```yaml
   on:
     pull_request:
       branches: [main, preprod]
       types: [opened, synchronize, reopened, ready_for_review]
   ```

2. **Always add a `concurrency` block with `cancel-in-progress: true`** for test/
   lint/build workflows, so pushing a new commit cancels the superseded run:
   ```yaml
   concurrency:
     group: <workflow>-${{ github.event.pull_request.number || github.ref }}
     cancel-in-progress: true
   ```
   **Exception:** deploy workflows keep `cancel-in-progress: false` — never kill a
   running deploy. Serialize them instead (same `group`, cancel false).

3. **Skip drafts:** `if: github.event.pull_request.draft == false` (plus a
   `workflow_dispatch` escape hatch) on each job.

4. **Cache dependencies.** `actions/setup-node@v4` with `cache: npm`, and `npm ci`
   (never `npm install`). Cache Playwright/Cypress browser binaries with
   `actions/cache@v4` on `~/.cache/ms-playwright`.

5. **`paths:` / `paths-ignore:` filters — but ONLY on non-required checks.** A
   required status check that is skipped by a path filter leaves the PR stuck on
   "Expected — waiting for status" forever. Check branch protection first
   (`gh api repos/<org>/<repo>/branches/<branch>/protection/required_status_checks`).
   For required checks, gate *steps* inside the job (e.g. `dorny/paths-filter`)
   instead of the trigger, so the job still reports success.

6. **Don't duplicate work across workflows.** Unit tests, lint, typecheck, and
   smoke tests for one repo belong in *one* PR-gate workflow sharing a single
   checkout + cached install — not separate files that each reinstall deps.

7. **Scheduled crons: pick the slowest cadence that still works.** A KB/doc sync
   or content job almost never needs sub-hourly latency. Add `workflow_dispatch`
   for on-demand runs instead of a tight `*/15`/`*/30` schedule. **Pause the
   schedule entirely when the downstream job is known-broken** (e.g. depleted API
   credits) — a cron that no-ops still bills a runner every fire.

8. **Stay on `ubuntu-*` runners.** macOS bills 10× and Windows 2×. Android/most
   builds run fine on Linux; only reach for macOS when you genuinely need Xcode.

## Checklist when reviewing a workflow change

- [ ] `pull_request` trigger (not bare `push`), draft-skipped
- [ ] `concurrency` + `cancel-in-progress: true` (false only for deploys)
- [ ] `npm ci` + dependency & browser caching
- [ ] No path filter on a *required* check
- [ ] Not re-running something another workflow already does
- [ ] Crons at the slowest acceptable cadence; paused if downstream is down
- [ ] `ubuntu-*` runner
