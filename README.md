# Fork Project Workflow

Automated tooling for forking third-party GitHub Actions into your organization with upstream sync, tag monitoring, and security scanning.

## Why

When your CI/CD pipelines depend on third-party GitHub Actions, you're exposed to supply chain attacks -- tag mutations, compromised maintainer accounts, or malicious code injection. Forking the Action and pinning to reviewed tags eliminates this risk.

This repo provides `fork-action.sh` to bootstrap forks with sync infrastructure, and thin caller workflows that reference centralized [reusable workflows](https://github.com/SamFleming-TylerTech/fork-sync-shared-workflow) for ongoing maintenance.

## How It Works

1. **`fork-action.sh`** (one-time setup) -- Creates the fork, caller workflows, manifest, and branch structure
2. **Sync Upstream** (weekly cron) -- Fast-forward merges the `upstream-tracking` branch with upstream's default branch, opens a PR for review
3. **Sync Tags** (weekly cron) -- Detects new upstream releases, tag mutations, and deletions
4. **Security Scan** (on PR) -- Runs dependency review, CodeQL analysis, and posts a diff summary

## Quick Start

```bash
# Fork a new action into your org
./fork-action.sh acme/deploy-action --tag v7.0.8

# Fork into a specific org/user
./fork-action.sh acme/deploy-action --tag v1.3.0 --org SamFleming-TylerTech

# Add sync infra to an existing fork
./fork-action.sh acme/deploy-action --existing --org SamFleming-TylerTech

# Update sync infra on an existing fork
./fork-action.sh acme/deploy-action --existing --force-update --org SamFleming-TylerTech
```

## What fork-action.sh Creates

In the fork repository:

```
.github/workflows/
  sync-upstream.yml    # Caller -> reusable sync workflow
  sync-tags.yml        # Caller -> reusable tag monitor
  security-scan.yml    # Caller -> reusable security scan
FORK_MANIFEST.json     # Upstream provenance and sync state
CODEOWNERS             # Protects sync infrastructure files
```

Each workflow is a thin caller that delegates to reusable workflows in `fork-sync-shared-workflow` via the `@v1` floating tag. Bug fixes and improvements propagate to all forks automatically.

## Options

| Flag | Description |
|------|-------------|
| `--org <org>` | Target GitHub org or user (default: `tyler-technologies-oss`) |
| `--tag <tag>` | Pin a specific upstream tag (e.g., `v7.0.8`) |
| `--existing` | Operate on an already-forked repo |
| `--force-update` | Overwrite existing sync infrastructure (use with `--existing`) |
| `--templates-repo <r>` | Central templates repo (default: `SamFleming-TylerTech/fork-sync-shared-workflow`) |
| `--templates-ref <ref>` | Central templates ref/tag (default: `v1`) |

Environment variables `FORK_ORG`, `TEMPLATES_REPO`, and `TEMPLATES_REF` can also be used.

## Prerequisites

- `gh` CLI (authenticated)
- `git`
- `jq`

## Repository Layout

```
fork-action.sh                              # Main bootstrap script
.github/workflows/
  caller-sync-upstream.yml                  # Caller template (deployed to forks)
  caller-sync-tags.yml                      # Caller template
  caller-security-scan.yml                  # Caller template
templates/
  FORK_MANIFEST.json                        # Manifest template
  CODEOWNERS                                # CODEOWNERS template
docs/
  workflow-migration-example.yml            # Before/after migration example
```

## Related

- [fork-sync-shared-workflow](https://github.com/SamFleming-TylerTech/fork-sync-shared-workflow) -- Reusable workflows referenced by caller templates
