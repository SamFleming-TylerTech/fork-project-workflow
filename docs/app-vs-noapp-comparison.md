# No-App vs App Mode Comparison

Comparison based on clean parity tests using identical upstreams (`uprightbass360/parity-test-app` and `uprightbass360/parity-test-noapp`) forked into `SamFleming-TylerTech`.

## Artifacts Created

Both modes produce identical artifacts from the same upstream changes (new commit, tag mutation, new release, tag deletion):

| Artifact | App Mode | No-App Mode |
|----------|----------|-------------|
| Sync PR | PR created by GitHub App bot | PR created by `github-actions[bot]` |
| PR body (diff stats, security files, checklist) | Identical | Identical |
| Diff summary PR comment | Posted | Posted |
| Dependency review PR comment | Posted (by action natively) | N/A (not available in `workflow_run` context) |
| PR check runs (linked, clickable) | `scan/dependency-review`, `scan/codeql`, `scan/diff-summary` | `security-scan`, `security-scan/dependency-review`, `security-scan/diff-summary` |
| `security-scan` branch protection check | Native workflow check | Commit status + check run via `report-status` job |
| Tag mutation issue | Created with `security-alert` label | Identical |
| New release issue | Created with `upstream-release` label | Identical |
| Tag deletion issue | Created with `needs-security-review` label | Identical |

## Behavioral Differences

| Aspect | App Mode | No-App Mode |
|--------|----------|-------------|
| **Secrets required** | `FORK_SYNC_APP_ID` + `FORK_SYNC_APP_PRIVATE_KEY` | None |
| **Setup complexity** | GitHub App creation, installation, per-repo secrets | Zero additional setup |
| **Workflows deployed** | 3 (`sync-upstream`, `sync-tags`, `security-scan`) | 3 (`sync-upstream`, `sync-tags`, `security-scan`) |
| **Security scan trigger** | `on: pull_request` (triggered by PR creation) | `on: workflow_run` (triggered after sync-upstream completes) |
| **Security scan runs per sync** | Multiple (1 per PR event, earlier runs cancelled by concurrency) | 1 (single `workflow_run` event) |
| **PR author** | GitHub App bot (e.g., `repo-fork-test`) | `github-actions[bot]` |
| **Scan jobs** | `dependency-review`, `codeql`, `diff-summary` | `dependency-review`, `diff-summary` (CodeQL removed -- irrelevant for composite actions) |
| **dependency-review** | Runs in PR context, posts its own comment | Runs with explicit `base-ref`/`head-ref`, no native comment |
| **Branch protection check** | `security-scan` (native workflow check) | `security-scan` (commit status + check run via API) |
| **PR checks tab** | Workflow runs linked natively | Check runs created via API -- same clickable experience |
| **Cancelled checks visible** | Yes (earlier concurrency-cancelled runs show on PR) | No (single run, no cancellations) |
| **Centralized updates** | All 3 workflows are thin callers -- updates propagate automatically | `sync-upstream` and `security-scan` are standalone -- use `--force-update` to propagate |
| **sync-tags updates** | Thin caller -- auto-updates | Thin caller -- auto-updates |

## How No-App Workflow Chaining Works

The key challenge with `GITHUB_TOKEN` is that PRs it creates don't trigger `on: pull_request` workflows. No-app mode solves this with `workflow_run`:

```
1. sync-upstream.yml runs (cron or manual trigger)
2. Creates PR using GITHUB_TOKEN
3. Workflow completes successfully
4. GitHub fires workflow_run event
5. security-scan.yml triggers automatically
6. find-pr job discovers the open sync PR
7. dependency-review and diff-summary run in parallel
8. report-status posts check runs + commit status on PR head SHA
9. Branch protection check is satisfied
```

## PR Checks Tab Comparison

### App Mode

The `on: pull_request` trigger fires on PR open and each push (synchronize), producing multiple workflow runs. The concurrency group cancels earlier runs, but cancelled check entries remain visible:

```
Security Scan (on: pull_request)     -- 3 runs triggered
  scan / dependency-review           -- 2 cancelled, 1 success
  scan / codeql                      -- 2 cancelled, 1 success
  scan / diff-summary                -- 2 cancelled, 1 success
Code scanning results / CodeQL       -- success
```

### No-App Mode

The `workflow_run` trigger fires exactly once when sync-upstream completes, producing a single clean run. Check runs are created via the API to appear linked on the PR:

```
Security Scan (workflow_run)         -- 1 run
  security-scan                      -- success (check run + commit status)
  security-scan/dependency-review    -- success (check run)
  security-scan/diff-summary         -- success (check run)
```

## Key Trade-off

The **app mode** gives fully centralized maintenance -- all logic lives in shared reusable workflows and propagates to all forks automatically via the `@v1` floating tag. It also runs CodeQL analysis and gets native dependency-review PR comments.

The **no-app mode** eliminates all secret management and GitHub App overhead. The trade-off is that `sync-upstream.yml` and `security-scan.yml` are standalone workflows (not thin callers), so changes require `--force-update` to propagate. However, `sync-tags.yml` remains a thin caller and auto-updates. The checks tab is cleaner (single run, no cancelled entries).

For most teams, the no-app mode is the better default -- zero setup, no secrets to manage, cleaner PR checks, and the workflow logic rarely changes once stable.
