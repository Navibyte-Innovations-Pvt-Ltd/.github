# CLAUDE.md

## What is this?

Central `.github` repo for **Navibyte-Innovations-Pvt-Ltd** GitHub org. Two jobs:

1. **Org profile README** — `profile/README.md` renders at https://github.com/Navibyte-Innovations-Pvt-Ltd
2. **Reusable CI workflows + caller stub sync** — `.github/workflows/` holds org-wide reusable workflows; `.github/caller-templates/` holds caller stubs that `sync-callers.yml` fans out to every non-archived repo in org every 6 hours.

## Architecture

```
profile/README.md                              # org profile page content
.github/workflows/
  claude-reusable.yml                          # reusable @claude mention handler
  claude-auto-fix-reusable.yml                 # reusable auto-fix on issue open
  sync-callers.yml                             # cron fan-out to org repos
.github/caller-templates/
  claude.yml                                   # caller stub → synced to every repo's .github/workflows/claude.yml
  claude-auto-fix.yml                          # caller stub → synced to every repo's .github/workflows/claude-auto-fix.yml
```

## Key rules

- **Caller templates are source of truth.** Edit `.github/caller-templates/*.yml` → sync workflow pushes changes to all org repos. Never edit individual repo `.github/workflows/claude.yml` — gets overwritten.
- **Reusable workflows live here.** All org repos reference `Navibyte-Innovations-Pvt-Ltd/.github/.github/workflows/*-reusable.yml@main` via `uses:`.
- **Runner:** `self-hosted` (not GitHub-hosted).
- **Sync cadence:** every 6 hours via cron. Manual trigger via `workflow_dispatch`.
- **Branch-protection fallback:** `sync-callers.yml` opens PR `chore/sync-claude-callers` when direct push blocked.

## Commands

```bash
# Trigger sync manually
gh workflow run sync-callers.yml --repo Navibyte-Innovations-Pvt-Ltd/.github

# View last sync run
gh run list --workflow=sync-callers.yml --repo Navibyte-Innovations-Pvt-Ltd/.github --limit 3

# Preview profile README
open profile/README.md
```

No build / no tests — pure YAML + Markdown repo.

## Common tasks

### Update org profile README
1. Edit `profile/README.md`.
2. Commit + push `main`. Renders immediately on org page.

### Change Claude caller stub for all org repos
1. Edit `.github/caller-templates/claude.yml` or `claude-auto-fix.yml`.
2. Commit + push `main`.
3. `sync-callers.yml` fires on push → fans out within minutes.
4. Verify: `gh run list --workflow=sync-callers.yml --limit 1`.

### Change reusable workflow logic
1. Edit `.github/workflows/claude-reusable.yml` / `claude-auto-fix-reusable.yml`.
2. Commit + push `main` → all caller repos pick up next run (pinned to `@main`).

### Add new file to fan-out
1. Add source in `.github/caller-templates/<name>.yml`.
2. Add `sync_file` call in `sync-callers.yml` `Sync each repo` step.
3. Test with `workflow_dispatch`.

## Gotchas

- `GH_TOKEN` secret must have `contents:write` + `pull-requests:write` across org.
- Targets = non-archived, non-fork, non-`.github` repos — filter lives in `sync-callers.yml` `List target repos` step.
- base64 encoding uses `-w0` (Linux) with `-i` fallback (mac). Self-hosted runner is Linux, so `-w0` primary.
- `sha` required for updates; omit for creates. Script handles both.
- If caller stub file content matches sha256 of remote → `SKIP`. Idempotent.

## Release

No versioning. No tags. `main` is live. Push = deploy.
